# 📘 Глава 1. Core Python (Уровень Архитектора)
## Тема 1.1. Управление памятью: Reference Counting, Garbage Collector, поиск утечек

### Введение: Контекст международных стандартов
Согласно актуальному стандарту **ISO/IEC 25010:2023** (System and software quality models), одним из ключевых атрибутов качества системы является **Performance Efficiency** (Эффективность производительности), которая включает подхарактеристику **Resource Utilization** (Использование ресурсов). Для SDET это означает, что тестовый фреймворк сам по себе является программным продуктом, который обязан эффективно расходовать ОЗУ (RAM) и не приводить к деградации серверов при длительных нагрузочных или E2E-прогонах.

В Python программист не выделяет память вручную (как `malloc` / `free` в C/C++), но непонимание работы интерпретатора CPython приводит к «медленным» утечкам памяти (Memory Leaks).

---

### Часть 1. Как CPython управляет памятью «под капотом»

CPython использует многоуровневую архитектуру выделения памяти. Операционная система выделяет память интерпретатору, а внутри него работает собственный менеджер (Python Memory Manager) и специальный аллокатор **`pymalloc`**, оптимизированный для быстрых операций с небольшими объектами (до 512 байт).

Однако очистка этой памяти базируется на двух фундаментальных механизмах:

#### 1. Подсчет ссылок (Reference Counting) — Основной механизм
Каждый объект в Python содержит скрытое поле, в котором хранится счетчик ссылок (Reference count) — количество переменных или структур данных, которые в данный момент указывают на этот объект.
*   Как только вы создаете ссылку `x = [1, 2, 3]`, счетчик равен 1.
*   Если вы передаете этот список в функцию или присваиваете другой переменной `y = x`, счетчик увеличивается.
*   При удалении переменной (`del y`) или выходе её из области видимости (завершение функции), счетчик уменьшается.
*   **Итог:** Как только счетчик достигает **0**, `pymalloc` *моментально* уничтожает объект и освобождает память.

**Преимущества:** Детерминированность и мгновенная очистка. Никаких пауз в работе программы (как "Stop-the-world" в Java).
**Недостаток:** Подсчет ссылок абсолютно бессилен против **циклических ссылок (Cyclic References)**.

Работу второго механизма разберём чуть ниже.

---

### Часть 2. Сборщик мусора (Generational Garbage Collector)

Если объект `A` ссылается на объект `B`, а объект `B` ссылается обратно на `A` (например, двусвязный список или графы), их счетчики ссылок никогда не достигнут нуля, даже если основная программа потеряла к ним доступ (`del A`, `del B`). Для спасения от таких ситуаций в Python существует **Циклический Сборщик Мусора (Cyclic GC)** из модуля `gc`.

#### Механизм поколений (Generational GC)
Чтобы не сканировать всю память (что сильно тормозит тесты), GC делит все объекты на **3 поколения (Generations)**, основываясь на гипотезе: *"Молодые объекты умирают быстро, старые живут долго"*:
*   **Generation 0:** Все новые объекты попадают сюда.
*   **Generation 1:** Объекты, пережившие одну чистку нулевого поколения.
*   **Generation 2:** Долгоживущие объекты (например, глобальные конфигурации, синглтоны драйвера, кэши сессии).

Сборщик запускается автоматически, когда количество выделений памяти (allocations) превышает количество освобождений (deallocations) на определенный порог (threshold). Он ищет изолированные «острова» объектов, ссылающихся только друг на друга, и уничтожает их.

---

### Часть 3. Утечки памяти (Memory Leaks) в Python

В языках с GC «утечка памяти» означает не потерю указателя (как в C), а **непреднамеренное удержание ссылок (Unintentional Retention)**. Сборщик мусора не удалит объект, если в программе осталась хоть одна активная ссылка на него.

**Типичные причины утечек в тестовых фреймворках (SDET Focus):**
1.  **Глобальные словари и списки (Unbounded Caches):** Если вы сохраняете ответы API в глобальный словарь `api_responses_history` для последующего использования в отчете, но никогда не очищаете его, память будет расти линейно.
2.  **Замыкания (Closures) и Event Handlers:** Подписки на события (например, коллбеки при получении MQTT сообщения), которые не были отписаны перед удалением объекта.
3.  **Ошибки фикстур Pytest (Session Scope):** Создание тяжелых объектов (например, дампов БД) в фикстуре `scope="session"`, которые не удаляются на этапе Teardown (`yield`), из-за чего они висят в ОЗУ до конца прогона всех 5000 тестов.
4.  **Проблемы с методом `__del__`:** В старых версиях Python наличие кастомного метода `__del__` в объектах, образующих цикл, делало их "несобираемыми" (Uncollectable). В современных (Python 3.4+) это решено, но плохая логика в `__del__` всё еще может блокировать освобождение ресурсов (например, открытых сокетов).

---

### Часть 4. Инструменты детектирования (Стек 2026 года)

В современной разработке (2025–2026) инженеры используют три основных инструмента для локализации утечек.

#### 1. Memray (The Game Changer от Bloomberg)
**Memray** — это индустриальный стандарт профилирования CPython. В отличие от старых библиотек, он имеет минимальный overhead, работает в production-окружении и поддерживает нативные C-расширения (что критично, если ваши тесты используют `numpy` или `pandas` для валидации данных).
*   **Flamegraphs (Пламенные графы):** Позволяет визуально определить, какая именно функция (до строчки кода) аллоцировала больше всего ОЗУ.
*   **Live Tracking (TUI):** Вы можете подключиться к работающему процессу Pytest (`memray attach <PID>`) и смотреть за потреблением в реальном времени в терминале.
*   **Pytest-плагин:** Имеет встроенную интеграцию (`pytest-memray`), которая автоматически проваливает (fail) тесты, если они выходят за заданный лимит памяти.

#### 2. Tracemalloc (Встроенный инструмент)
Стандартная библиотека CPython. Делает снимки (snapshots) состояния памяти.
Идеально для A/B сравнения.
```python
import tracemalloc

tracemalloc.start()
# ... выполнение тяжелого API теста ...
snapshot1 = tracemalloc.take_snapshot()
# ... выполнение еще 10 тестов ...
snapshot2 = tracemalloc.take_snapshot()

# Сравниваем снимки, чтобы найти, какие объекты "приросли"
top_stats = snapshot2.compare_to(snapshot1, 'lineno')
for stat in top_stats[:5]:
    print(stat) # Покажет точный файл и строку, где произошла утечка
```

#### 3. Objgraph
Если `tracemalloc` показывает *где* выделена память, то **`objgraph`** показывает *почему* объект не удаляется. Он рисует граф ссылок, позволяя визуально найти ту самую глобальную переменную, которая удерживает "мертвый" объект.

---

### Часть 5. Best Practices & Bad Practices (SDET Контекст)

#### ❌ Bad Practices (Как делать нельзя)
1.  **Агрессивный вызов `gc.collect()`:** Попытки вставлять принудительный запуск сборщика мусора в `teardown` каждого теста. Это скрывает симптомы (маскирует кривую архитектуру), сжигает процессорное время (CPU) и замедляет пайплайн.
2.  **Чтение огромных файлов в память (Slurping):**
    ```python
    # BAD: Выгружает 2GB лог-файл прямо в ОЗУ
    with open("app_log.txt") as f:
        data = f.readlines() 
    ```
3.  **Бесконечные кэши в долгих прогонах:** Использование декоратора `@lru_cache(maxsize=None)` для функций, генерирующих тестовые данные. В долгих (Endurance) тестах этот кэш сожрет всю память сервера.

#### ✅ Best Practices (Стандарт 2026 года)
1.  **Потоковая обработка (Streaming & Generators):**
    Всегда используйте генераторы (`yield`) для обработки больших JSON-ответов или логов. Объекты создаются и сразу уничтожаются Reference Counter-ом, не дожидаясь GC.
    ```python
    # GOOD: Память потребляется O(1)
    with open("app_log.txt") as f:
        for line in f:
            if "ERROR" in line:
                yield line
    ```
2.  **Использование Weak References (Слабых ссылок):**
    Если вам нужно реализовать сложный кэш для сессии, используйте модуль `weakref`. Слабая ссылка позволяет ссылаться на объект, но не увеличивает его *Reference Count*. Если других ссылок нет, объект будет уничтожен.
3.  **Идемпотентный Teardown:**
    Для каждого класса PageObject или API-клиента, открывающего сессии (например, `httpx.AsyncClient` или Playwright `BrowserContext`), используйте паттерн Контекстного менеджера (`with ...`) или гарантированное закрытие в `finally`. Ненадежно закрытый сетевой сокет удерживает память на уровне ядра ОС.
4.  **Мониторинг процесса (psutil):**
    В нагрузочных тестах или скриптах-демонах встраивайте алерты на основе RSS (Resident Set Size). Если процесс Pytest выходит за лимит в 1GB, выводите Warning в CI/CD.

---

### Резюме
В 2026 году Senior SDET обязан понимать, что память в Python не исчезает «по волшебству». Основной механизм очистки — **Подсчет ссылок** — справляется с 90% работы быстро и незаметно, но оставшиеся 10% (циклические ссылки и "осевшие" в кэшах данные) требуют работы **Сборщика мусора**. Использование инструментов типа **Memray** и грамотный дизайн архитектуры тестов гарантируют, что ваши фреймворки будут соответствовать стандартам `ISO/IEC 25010:2023 (Resource Utilization)`, обеспечивая стабильность CI/CD пайплайнов любой сложности.

Sources:
1. [A guideline for software architecture selection based on ISO 25010 quality related characteristics](https://www.researchgate.net/publication/309820148_A_guideline_for_software_architecture_selection_based_on_ISO_25010_quality_related_characteristics)
2. [Designing LLM-based Multi-Agent Systems for Software Engineering Tasks: Quality Attributes, Design Patterns and Rationale](https://arxiv.org/html/2511.08475v2)
3. [New Model for Defining and Implementing Performance Tests](https://www.mdpi.com/1999-5903/16/10/366)
4. [Memory Profiling CPython Applications with Memray](https://mienxiu.com/memory-profiling-with-memray/)
5. [A Deep Dive Into Python’s Memory Management and Garbage Collection](https://medium.com/pythoneers/a-deep-dive-into-pythons-memory-management-and-garbage-collection-9ba36b77e40b)
6. [Garbage Collection in Python: A Tutorial](https://builtin.com/articles/garbage-collection-in-python)
7. [Understanding Python Memory and Garbage Collection Through Hands-On Experiments](https://dev.to/kfir-g/understanding-python-memory-and-garbage-collection-through-hands-on-experiments-4g0p)
8. [How to Handle Memory Leaks in Python](https://oneuptime.com/blog/post/2025-01-06-python-memory-leak-debugging/view)
9. [Python’s Garbage Collector Explained: The Hidden Hero of Memory Management](https://python.plainenglish.io/pythons-garbage-collector-explained-the-hidden-hero-of-memory-management-85583ebb74cf)
10. [Garbage Collection in Python](https://www.geeksforgeeks.org/python/garbage-collection-python/)
11. [Python’s Garbage Collector Explained: The Hidden Hero of Memory Management](https://python.plainenglish.io/pythons-garbage-collector-explained-the-hidden-hero-of-memory-management-85583ebb74cf)
12. [How to Debug Memory Leaks in Python](https://oneuptime.com/blog/post/2026-01-24-debug-memory-leaks-python/view)
13. [Top 5 Python Memory Leak Detection Techniques for Long‑Running Services](https://www.techbuddies.io/2026/02/21/top-5-python-memory-leak-detection-techniques-for-long-running-services/)
14. [Ray - Profiling](https://docs.ray.io/en/latest/ray-observability/user-guides/profiling.html)
15. [bloomberg - memray](https://github.com/bloomberg/memray)
16. [Python memory leaks](https://codemia.io/knowledge-hub/path/python_memory_leaks_closed)
