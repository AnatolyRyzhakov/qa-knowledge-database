# 📘 Глава 1. Core Python (Уровень Архитектора)
## Тема 1.4. Метапрограммирование: Декораторы, Контекстные менеджеры и Magic Methods

### Введение: Контекст международных стандартов
Согласно стандарту **ISO/IEC 25010:2023**, атрибут качества **Reliability** (Надежность) включает подхарактеристику **Recoverability** (Восстанавливаемость) — способность системы восстанавливать свою работу после сбоя. 
В обновленном стандарте тестирования **ISTQB Advanced Level Test Automation Engineering (TAE v2.0)** (выпущенном в конце 2024 года и ставшем стандартом в 2025-2026) огромный фокус делается на обработку исключений (Error Recovery) и устойчивость автотестов к нестабильности тестовых сред. 

Чтобы не засорять бизнес-логику автотестов конструкциями `try/except` на каждом шагу, Senior SDET использует метапрограммирование: Декораторы и Контекстные менеджеры.

---

### Часть 1. Декораторы и Паттерн `@retry` (Resilience 2026)

Декоратор — это функция высшего порядка, которая принимает другую функцию, оборачивает её в дополнительную логику и возвращает модифицированную версию. Это идеальная реализация принципа **Separation of Concerns (Разделение ответственности)**.

#### Эволюция `@retry` в автотестировании
Проблема "плавающих" тестов (Flaky tests) часто связана с кратковременными сетевыми сбоями (Transient errors) или Rate Limits (ограничениями частоты запросов). В 2026 году писать простую попытку перезапуска (retry) уже недостаточно. Индустрия требует **Exponential Backoff с Jitter (экспоненциальную задержку с фактором случайности)**, чтобы при массовом падении тестов их одновременный перезапуск не устроил DDoS-атаку на ваш же тестовый сервер.

**Пример архитектурно правильного асинхронного декоратора:**

```python
import asyncio
import random
from functools import wraps
import httpx

def async_retry(max_retries=3, base_delay=1.0, exceptions=(httpx.RequestError,)):
    """
    Асинхронный декоратор с экспоненциальной задержкой и Jitter.
    """
    def decorator(func):
        @wraps(func) # КРИТИЧЕСКИ ВАЖНО: Сохраняет оригинальное имя и docstring функции
        async def wrapper(*args, **kwargs):
            retries = 0
            while retries < max_retries:
                try:
                    return await func(*args, **kwargs)
                except exceptions as e:
                    retries += 1
                    if retries == max_retries:
                        raise e # Пробрасываем ошибку дальше, если лимит исчерпан
                    
                    # Exponential Backoff + Jitter
                    delay = (base_delay * 2 ** (retries - 1)) + random.uniform(0, 0.5)
                    print(f"[Retry] Ошибка {e}. Попытка {retries}/{max_retries}. Ждем {delay:.2f} сек.")
                    await asyncio.sleep(delay)
        return wrapper
    return decorator

# Применение в тестах (код теста остается идеально чистым)
@async_retry(max_retries=5, exceptions=(httpx.TimeoutException,))
async def fetch_user_data(api_client, user_id):
    return await api_client.get(f"/api/users/{user_id}")
```

---

### Часть 2. Контекстные менеджеры (`__enter__` и `__exit__`)

Контекстные менеджеры используются для управления ресурсами по паттерну **RAII** (Resource Acquisition Is Initialization). Их главная задача — гарантировать, что блок очистки (Teardown) будет выполнен **даже если внутри теста произошла критическая ошибка (Exception)**.

#### Синхронные менеджеры
Классический контекстный менеджер реализует методы `__enter__()` (настройка/выделение) и `__exit__(exc_type, exc_val, exc_tb)` (очистка). Если `__exit__` возвращает `True`, ошибка "проглатывается" (что в автотестах обычно является антипаттерном).

#### Асинхронные менеджеры (`__aenter__` и `__aexit__`) — Стандарт для I/O
Поскольку 90% современного тестирования — это сетевые запросы (Playwright, базы данных, API), Senior SDET обязан работать с `async with`. Асинхронные методы позволяют применять `await` прямо на этапах Setup и Teardown.

**Пример: Безопасное управление сессией базы данных для тестов:**
```python
import asyncio
import asyncpg

class AsyncTestDBContext:
    def __init__(self, db_url):
        self.db_url = db_url
        self.conn = None

    async def __aenter__(self):
        # Setup: устанавливаем соединение асинхронно
        self.conn = await asyncpg.connect(self.db_url)
        # Начинаем транзакцию, чтобы изолировать тестовые данные
        self.tr = self.conn.transaction()
        await self.tr.start()
        return self.conn

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        # Teardown: Всегда откатываем (Rollback) изменения, 
        # чтобы база осталась чистой после теста
        if self.tr:
            await self.tr.rollback()
        if self.conn:
            await self.conn.close()
            
# Использование в Pytest (гарантия изоляции)
async def test_create_user():
    async with AsyncTestDBContext("postgresql://user:pass@localhost/testdb") as db:
        await db.execute("INSERT INTO users (name) VALUES ('Test')")
        # Как только мы выходим из блока (даже при падении assert),
        # сработает __aexit__, сделает rollback() и закроет соединение[11]
```
*(Альтернатива без ООП: использование декоратора `@asynccontextmanager` из библиотеки `contextlib` для генераторов).*

---

### Часть 3. Magic Methods (Dunder Methods) во фреймворках

"Магические" методы (с двойным подчеркиванием) позволяют переопределять встроенное поведение объектов Python. Senior SDET использует их для создания интуитивно понятных "Pythonic" DSL (Domain Specific Language) библиотек для своей команды.

Наиболее полезные методы для тестирования:
1.  **`__new__(cls)`:** Отвечает за *создание* инстанса. Используется для реализации паттерна **Singleton** (например, чтобы гарантировать, что во всем прогоне тестов существует только один экземпляр глобального логгера или конфигуратора).
2.  **`__call__(self)`:** Позволяет вызывать объект класса так, будто это функция. Часто используется в паттерне Screenplay или сложных моках (Mock objects).
3.  **`__getitem__(self, key)`:** Переопределяет доступ по индексу (как в словарях).
4.  **`__getattr__(self, name)`:** Динамический перехват несуществующих методов. Это основа библиотеки `unittest.mock.MagicMock`.

**Пример из практики (Создание элегантного API клиента):**
```python
class SmartAPIClient:
    def __init__(self, base_url):
        self.base_url = base_url

    # Позволяет обращаться к эндпоинтам через точку: client.users.get()
    def __getattr__(self, endpoint):
        return SmartAPIClient(f"{self.base_url}/{endpoint}")

    def __call__(self, payload=None):
        print(f"Отправка GET запроса на {self.base_url} с данными {payload}")
        return {"status": 200}

# Использование Junior QA (очень читаемый код):
client = SmartAPIClient("https://api.test.com")
response = client.v1.users.profiles(payload={"id": 123}) 
# Динамически сформирует URL: https://api.test.com/v1/users/profiles
```

---

### Часть 4. Best Practices & Bad Practices

#### ❌ Bad Practices (Антипаттерны Архитектора)
1.  **Потеря метаданных функции:** Написание декоратора без использования `@functools.wraps(func)`. Это стирает оригинальное имя функции (`__name__`) и docstring, из-за чего Pytest перестает понимать, что это тест-кейс, а Allure-отчеты генерируют пустые имена шагов.
2.  **Блокировка Event Loop в `__enter__`:** Использование синхронных методов (например, `time.sleep` или синхронных запросов к БД) внутри асинхронного `__aenter__`. Это убьет всю производительность асинхронного тестового пайплайна.
3.  **"Глотание" исключений:** Возврат `True` из метода `__exit__` или `__aexit__` без явной архитектурной необходимости. Это скроет факт падения теста от Pytest, и CI/CD всегда будет "зеленым", даже если продукт не работает.

#### ✅ Best Practices (Стандарты 2026)
1.  **Типизация декораторов (Type Hinting):** В 2026 году (с развитием `mypy` и TypeScript-подобного подхода) все декораторы должны быть строго типизированы с помощью `ParamSpec` и `TypeVar`. Это гарантирует, что IDE (VS Code / PyCharm) будет правильно подсказывать аргументы оборачиваемой функции.
2.  **Разделение логики:** Декоратор `@retry` не должен содержать бизнес-логику (например, авторизацию). Если при неудачном запросе нужно обновить токен — это задача Interceptor'ов на уровне HTTP-клиента (например, Middleware в `httpx`), а не декоратора теста.
3.  **ExitStack для динамических ресурсов:** Если количество контекстных менеджеров неизвестно заранее (например, тест динамически открывает от 1 до 5 браузерных вкладок), используйте `contextlib.AsyncExitStack`. Он гарантированно закроет все открытые ресурсы в обратном порядке при завершении теста.

---

### 📚 Sources (Источники и стандарты)
1.  **ISO/IEC 25010:2023** – *Systems and software engineering — Quality Requirements and Evaluation*. (Раздел: Reliability -> Recoverability).
2.  **ISTQB CTAL-TAE v2.0 (2024/2025 release)** – *Certified Tester Advanced Level - Test Automation Engineering*. (Глава: "Deployment Risks and Automation Mitigation Strategies" - внедрение отказоустойчивости в TAF).
3.  **Decodo (Jan 2026)** – *Retry Failed Python Requests in 2026*. (Best practices for using Custom Retry logic, HTTPAdapters, and Jitter in API testing).
4.  **DEV Community (Oct 2025)** – *Why You Should Care About Async Context Managers and Iterators*. (Глубокий разбор протоколов `__aenter__` и `__aexit__` для безопасного управления ресурсами).
5.  **CoderPad (2022-2026 updates)** – *A Guide to Python’s Secret Superpower: Magic Methods*. (Использование Dunder methods для создания детерминированных приложений и чистого API).
6.  **Qentelli (Sep 2025)** – *How Resilience Engineering Transforms QA Beyond Bug Detection*. (Переход от простого поиска багов к проверке устойчивости и восстановления: Shift-Left Resilience).
