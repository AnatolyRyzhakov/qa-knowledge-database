# 📘 Глава 1. Core Python (Уровень Архитектора)
## Тема 1.3. Продвинутые структуры и ленивые вычисления: Итераторы и Генераторы для обработки больших данных

### Введение: Контекст международных стандартов
Согласно стандарту **ISO/IEC 25010:2023** (Performance Efficiency -> **Resource Utilization** и **Capacity**), система должна сохранять стабильность при обработке пиковых объемов информации. Для процессов тестирования это регулируется **ISTQB Advanced Level Test Automation Engineer (CTAL-TAE)** в разделе *Test Data Management*. Если ваш фреймворк потребляет 16 GB RAM для валидации датасета в 10 GB — архитектура спроектирована неверно. 

Решение этой проблемы в Python — **Ленивые вычисления (Lazy Evaluation)**. Это концепция, при которой значения вычисляются (или считываются из сети/диска) строго в тот момент, когда они требуются, а не заранее.

---

### Часть 1. Анатомия итерации: Iterables vs Iterators
Для начала разведем понятия на уровне CPython:
*   **Iterable (Итерируемый объект):** Любой объект, реализующий магический метод `__iter__()`, который возвращает итератор. (Например, `list`, `dict`, `str`). Они хранят **все** свои элементы в памяти.
*   **Iterator (Итератор):** Объект с состоянием. Реализует методы `__iter__()` и `__next__()`. Он не хранит элементы, он знает только, как получить *следующий* элемент, и выбрасывает исключение `StopIteration`, когда элементы заканчиваются.

**Архитектурный смысл:** Итератор имеет сложность по памяти **$O(1)$**, независимо от того, обрабатывает он 10 записей или 10 миллиардов.

---

### Часть 2. Генераторы: Фабрики итераторов и управление состоянием
Генератор — это самый элегантный способ создания итераторов в Python. Любая функция, содержащая ключевое слово `yield`, становится генераторной функцией.

В отличие от `return`, который уничтожает локальные переменные (stack frame) функции, `yield` **замораживает (suspends)** состояние функции. При следующем вызове `next()` выполнение продолжается ровно с того же места, сохраняя все локальные переменные.

#### Продвинутые возможности: Data Pipelines (Конвейеры данных)
В 2026 году Senior SDET используют генераторы для построения Unix-подобных пайплайнов обработки данных, связывая их через `yield from`.

```python
# Эффективный конвейер для обработки терабайтных логов (Memory: ~10 MB)
def read_huge_log(file_path):
    with open(file_path, 'r') as f:
        for line in f:
            yield line

def filter_errors(lines):
    for line in lines:
        if "ERROR" in line or "CRITICAL" in line:
            yield line

def extract_tracebacks(error_lines):
    # Продвинутое делегирование генераторов
    for line in error_lines:
        yield from parse_traceback_logic(line)

# Запуск конвейера (вычисления не начнутся, пока мы не запросим данные)
log_stream = read_huge_log("/var/log/microservices_2026.log")
errors = filter_errors(log_stream)
tracebacks = extract_tracebacks(errors)

# Обработка происходит по 1 строке за раз (Лениво)
for tb in tracebacks:
    send_to_allure_report(tb)
```

#### Обратная связь: `.send()`, `.throw()`, `.close()`
Генераторы могут не только отдавать данные, но и принимать их, превращаясь в **корутины (coroutines)**.
С помощью `value = yield data` вы можете динамически отправлять управляющие команды прямо в работающий генератор (метод `gen.send(value)`). Это используется в сложных фикстурах для изменения состояния тестового окружения на лету.

---

### Часть 3. Асинхронные генераторы (The 2026 Standard)
С повсеместным переходом тестирования на асинхронные протоколы (gRPC стриминг, WebSockets, Server-Sent Events, HTTP/3 QUIC streams), классические синхронные генераторы стали "узким горлышком" (blocking I/O).

**PEP 525 (Asynchronous Generators)** позволяет использовать `yield` внутри `async def` и читать данные через `async for`.

**Пример из реальной практики SDET (Тестирование потока Kafka или SSE):**
```python
import asyncio
import httpx

async def stream_ai_model_responses(api_url: str):
    """Асинхронный генератор для чтения потоковых ответов LLM-модели (SSE)"""
    async with httpx.AsyncClient() as client:
        async with client.stream("GET", api_url) as response:
            async for chunk in response.aiter_text():
                if chunk:
                    yield chunk  # Лениво отдаем чанки по мере их прибытия по сети

async def test_llm_streaming_latency():
    # Мы не ждем окончания генерации (которая может занять 30 сек), 
    # а валидируем каждый токен мгновенно.
    async for token in stream_ai_model_responses("https://api.ai-service.com/stream"):
        assert not contains_hallucination(token)
        validate_latency_requirements()
```

---

### Часть 4. Индустриальный стандарт: Модуль `itertools`
Писать собственные циклы `for` с `yield` часто избыточно. В арсенале Senior SDET всегда есть стандартная библиотека `itertools`. Ее функции написаны на C, работают в обход некоторых проверок Python-машины и выполняются молниеносно.

*   `itertools.islice(iterable, start, stop)`: Ленивый срез. Никогда не делайте `huge_list[:1000]`, это создаст копию в памяти. Используйте `islice`.
*   `itertools.chain(*iterables)`: Объединяет несколько лог-файлов или датасетов в один непрерывный поток без выделения дополнительной памяти.
*   `itertools.groupby()`: Группировка потоковых данных на лету (например, логов по уровням критичности).

---

### Часть 5. Best Practices & Bad Practices

#### ❌ Bad Practices (Архитектурные антипаттерны)
1.  **Материализация генератора (The `list()` trap):**
    ```python
    # УБИЙСТВО ПАМЯТИ: Сборка всего ленивого генератора в список перед проверкой
    data_gen = generate_10_million_synthetic_users()
    users_list = list(data_gen) 
    assert len(users_list) == 10000000
    ```
    *Почему плохо:* Это полностью уничтожает смысл ленивых вычислений. Используйте агрегирующие функции (например, счетчик через цикл), если вам нужно узнать количество элементов.
2.  **Использование Генераторов для повторного чтения:**
    Генератор истощается (exhausted) после первого прохода. Если в тесте вам нужно пройтись по сгенерированному датасету дважды (например, сначала найти максимум, а потом вычислить среднее), генератор вернет пустой результат на втором проходе. *Решение:* использовать `itertools.tee()` для разветвления потока.
3.  **Забытый `yield` в фикстурах Pytest:** Использование `return` вместо `yield` при выдаче ресурсов БД. Это оставляет открытые коннекты (connection leaks), так как блок teardown (закрытия) никогда не будет вызван.

#### ✅ Best Practices (Стандарт 2026)
1.  **Fuzzing & Synthetic Data Generation:**
    Используйте бесконечные генераторы (`while True: yield data`) в связке с `itertools.islice` для Data-Driven тестирования. Это позволяет на лету скармливать тестам уникальные payload'ы, не сохраняя их на диск.
2.  **Память $O(1)$ для больших файлов:**
    Любой файл в Python уже является ленивым итератором по строкам. Никогда не используйте `.read()` или `.readlines()` для файлов > 50 MB.
3.  **Асинхронные фикстуры (Pytest-asyncio):**
    Все фикстуры, создающие сетевые ресурсы для тестов, должны использовать `async_generator`.
    ```python
    @pytest_asyncio.fixture
    async def db_connection():
        conn = await asyncpg.connect(DSN)
        yield conn  # Замораживаем, отдаем ресурс тесту
        await conn.close()  # Размораживаем после теста и безопасно закрываем
    ```

---

### 📚 Sources (Источники и стандарты)

1.  **ISO/IEC 25010:2023** – *Systems and software engineering — Systems and software Quality Requirements and Evaluation (SQuaRE)*. (Раздел 4.2.4: Performance Efficiency - Resource Utilization).
2.  **ISTQB Advanced Level Syllabus - Test Automation Engineer (v2024/2026 update)** – *Section 5.3: Test Data Management and Generation strategies* (Использование динамической генерации данных вместо статических дампов).
3.  **Python Enhancement Proposals (PEPs)**:
    *   *PEP 255 – Simple Generators* (Фундамент ленивых вычислений).
    *   *PEP 380 – Syntax for Delegating to a Subgenerator* (Введение конструкции `yield from` для построения пайплайнов).
    *   *PEP 525 – Asynchronous Generators* (Стандартизация `async def` + `yield` для современного I/O).
4.  **Python Official Documentation (3.14+)** – Разделы `itertools` (Functions creating iterators for efficient looping) и `asyncio` (Streams and Asynchronous Iterators).
5.  **"High Performance Python" (Advanced Practices 2025/2026)** – Использование генераторов для оптимизации кэша L1/L2 процессора за счет последовательного доступа к памяти (Data Locality).
