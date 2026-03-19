# 📘 Глава 4. Мастерство в Pytest (Фреймворк как экосистема)
## Тема 4.1. Управление состоянием: Продвинутые фикстуры, `yield`, scopes

### Введение: Контекст стандартов

Для Senior SDET Pytest — это не просто запускалка тестов (test runner), это мощнейший **IoC-контейнер (Inversion of Control)**, который берет на себя управление всем жизненным циклом тестового окружения.

В 2026 году (с переходом к сложным микросервисам и асинхронным протоколам) классический подход xUnit с методами `setUp()` и `tearDown()` окончательно признан устаревшим [1]. На смену ему пришли модульные, масштабируемые фикстуры.

Согласно стандарту **ISO/IEC 25010** (Reliability -> Resource Utilization), тестовый фреймворк не должен исчерпывать системные ресурсы (порты, ОЗУ, коннекты к БД). В тестировании (ISTQB Advanced TAE) это называется **Test Environment State Management**. Если ваши 500 тестов каждый раз заново поднимают Docker-контейнер с базой данных, прогон займет часы. Если поднять базу один раз на весь прогон, но не очищать ее, тесты начнут влиять друг на друга (State Leakage / Flaky tests) [4].

Pytest решает эту проблему через систему Фикстур (Fixtures) с разными областями видимости (Scopes) и механизмами безопасной очистки ресурсов.

---

### Часть 1. Архитектура Scopes (Области видимости)
В Pytest фикстура инстанцируется только тогда, когда она впервые запрашивается тестом, и уничтожается на основе её `scope` [1]. 

Senior SDET должен виртуозно жонглировать пятью уровнями [1][2]:
1.  **`scope="function"` (По умолчанию):** Фикстура создается и уничтожается для *каждого* тестового метода. Идеально для изоляции данных (например, генерация уникального `user_id` для каждого теста).
2.  **`scope="class"` / `scope="module"`:** Фикстура создается один раз для всего тестового класса или файла `.py`. Применяется для группировки тестов, требующих единого дорогого пред-условия (например, авторизация под ролью Admin для набора из 10 проверок Admin-панели).
3.  **`scope="package"`:** Фикстура живет на уровне директории (и всех ее подпапок).
4.  **`scope="session"`:** Фикстура создается *один раз* за весь запуск (test run).
    *Применение 2026:* Инициализация пула подключений к БД, запуск облачной Selenium/Playwright сессии, генерация глобального отчета Allure [2].

**🔥 Продвинутая техника: Динамический Scope**
В современных фреймворках Scope часто зависит от окружения (Local vs CI/CD). Pytest позволяет передавать функцию (callable) в параметр `scope` [1].
```python
def determine_scope(fixture_name, config):
    # Если тесты запущены в CI, используем 'session' для скорости. 
    # Если локально - 'function' для строгой изоляции при дебаге.
    if config.getoption("--env") == "ci":
        return "session"
    return "function"

@pytest.fixture(scope=determine_scope)
def db_connection():
    # ... логика подключения ...
```

---

### Часть 2. Механизм `yield` и гарантированный Teardown
Классический `return` в фикстурах не позволяет выполнить очистку (Teardown). Поэтому стандартом является использование генераторного паттерна с `yield` [1][2].

**Анатомия `yield` фикстуры:**
```python
@pytest.fixture(scope="session")
def mqtt_broker():
    # 1. SETUP (Инициализация)
    broker = start_docker_mqtt_broker()
    print("Брокер запущен")
    
    # 2. Передача управления в тест (инжект ресурса)
    yield broker 
    
    # 3. TEARDOWN (Очистка)
    print("Остановка брокера")
    broker.stop()
```

**Критически важный нюанс поведения (Вопрос с FAANG-интервью):**
*"Что произойдет с блоком Teardown, если внутри теста упадет `assert`?"*
*   **Ответ Senior'а:** Pytest **гарантированно** выполнит код после `yield` (Teardown), даже если тест провалился с ошибкой (Failed) [1]. Однако, если исключение возникло *до* `yield` (на этапе Setup), Pytest прервет выполнение и *не* будет вызывать Teardown этой фикстуры, так как ресурс не был успешно создан [1].

**Альтернатива: `request.addfinalizer()`**
Хотя `yield` читается легче, в сложных динамических фикстурах (когда ресурс может создаться, а может и нет) используют `addfinalizer`. Финализаторы исполняются в строгом порядке LIFO (последним пришел - первым ушел) [1].

---

### Часть 3. Асинхронные фикстуры и Event Loops (Стандарт 2026)
В современном Python I/O-операции (БД, API, Playwright) асинхронны. Плагин `pytest-asyncio` позволяет использовать `async def` для фикстур, решая проблему блокировки потоков [3].

**Проблема Event Loop:**
По умолчанию `pytest-asyncio` создает новый Event Loop (цикл событий) для каждого теста (Scope = function). Если ваша база данных инициализируется в фикстуре уровня `session`, вы получите ошибку `"Task attached to a different loop"`.

**Решение (Переопределение Event Loop):**
Senior SDET всегда явно управляет циклом событий, переопределяя встроенную фикстуру `event_loop` на уровень сессии [3].
```python
import pytest
import asyncio
import httpx

# 1. Поднимаем Event Loop на уровень всей сессии
@pytest.fixture(scope="session")
def event_loop():
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

# 2. Асинхронная фикстура сессионного уровня использует этот же Loop
@pytest.fixture(scope="session")
async def async_api_client():
    client = httpx.AsyncClient(base_url="https://api.test.com")
    yield client
    await client.aclose() # Неблокирующий безопасный teardown [3]
```

---

### Часть 4. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns (Как делать нельзя)
1.  **Мутация Session-фикстур (Shared State Mutation):** Если `scope="session"` фикстура возвращает список (например, кэш пользователей), а Тест №1 удаляет из него элементы, то Тест №2 упадет. Это классическая причина Flaky Tests [4]. Ресурсы уровня сессии должны быть **неизменяемыми (Immutable)**.
2.  **Злоупотребление `autouse=True`:** Этот флаг заставляет фикстуру выполняться "в фоне" для всех тестов, даже если они её не запрашивали. Это магия, которая убивает читаемость кода (явное лучше неявного) [2]. Использовать `autouse` можно только для глобальных логгеров или инициализации переменных окружения.
3.  **God Object `conftest.py`:** Сваливание всех фикстур проекта в один корневой файл `conftest.py`. Pytest поддерживает иерархию: фикстуры для API должны лежать в `tests/api/conftest.py`, а для UI — в `tests/ui/conftest.py`[1].

#### ✅ Best Practices (Стандарты 2026)
1.  **Fixture Composition (Композиция фикстур):** Фикстуры могут запрашивать другие фикстуры. Разделяйте логику.
    *   `fixture_db_connect` (подключается к БД).
    *   `fixture_generate_user(fixture_db_connect)` (использует коннект для создания юзера).
    *   `test_login(fixture_generate_user)` (использует готового юзера).
2.  **State Isolation (Изоляция состояния):** Если тесту нужна БД, фикстура должна использовать транзакции. В `yield` открывается транзакция, а в `teardown` всегда вызывается `rollback()` [3]. Так база данных останется девственно чистой для следующего теста, даже если текущий тест внес туда мусор.
3.  **Использование `request.node`:** Встроенная фикстура `request` позволяет получить доступ к метаданным выполняющегося теста (например, к его имени `request.node.name`). Senior SDET использует это для динамического нейминга файлов логов или скриншотов в `teardown` [1].

***

### 📚 Sources (Источники)
1. [pytest fixtures: explicit, modular, scalable — pytest documentation](https://docs.pytest.org/en/latest/explanation/fixtures.html)
2. [Supercharging Your Tests: Pytest Fixtures for Reusable Setup and Teardown — Dev.to](https://dev.to/bshadmehr/supercharging-your-tests-pytest-fixtures-for-reusable-setup-and-teardown-3a2b)
3. [Async Testing with Pytest: Mastering pytest-asyncio and Event Loops — Medium](https://medium.com/@connect.hashblock/async-testing-with-pytest-mastering-pytest-asyncio-and-event-loops-for-fastapi-and-beyond-37c613f1cfa3)
4. [Unit Testing Anti-patterns Catalogue / Flaky Tests — StackOverflow Engineering](https://stackoverflow.com/questions/333682/unit-testing-anti-patterns-catalogue)
