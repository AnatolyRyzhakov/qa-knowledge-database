# 📂 Module 1.1.3. Порождающие паттерны в автотестах (Builder, Singleton, Factory)

Привет! Двигаемся вглубь архитектуры. Порождающие паттерны (Creational Patterns) отвечают за **создание объектов**. В автоматизации тестирования мы постоянно что-то создаем: тестовые данные (огромные JSON-payloads), подключения к БД, экземпляры браузеров или API-клиенты. 

Если создавать эти объекты «в лоб», код быстро превратится в нечитаемую кашу из словарей и длинных списков аргументов. Лид/Архитектор использует паттерны, чтобы сделать создание объектов безопасным, читаемым и переиспользуемым.

Разберем топ-3 порождающих паттерна и то, как они применяются в Python в реалиях 2025–2026 годов.

---

## 1. Builder (Строитель): Генерация сложных тестовых данных

**Проблема:** Тебе нужно отправить POST-запрос на создание пользователя. У пользователя есть 20 полей (адрес, настройки, роли), но для конкретного теста нужно изменить только одно поле (например, `role="admin"`), а остальные заполнить дефолтными валидными данными. Использование огромных словарей `dict` или функций с 20 аргументами — это боль.

**Решение:** Паттерн Builder (в связке с Fluent Interface). Он позволяет конструировать объект шаг за шагом. В современном Python (3.11+) для этого идеально подходит использование `typing.Self` и `pydantic`.

### 💻 Пример: Modern Python Builder

```python
from pydantic import BaseModel, Field
from typing import Self
import uuid

# 1. Pydantic-модель (Схема данных, DTO)
class UserPayload(BaseModel):
    username: str = Field(default_factory=lambda: f"user_{uuid.uuid4().hex[:6]}")
    email: str = Field(default="test@example.com")
    role: str = Field(default="guest")
    is_active: bool = Field(default=True)

# 2. Builder для удобной настройки модели в тестах
class UserPayloadBuilder:
    def __init__(self):
        self._payload = UserPayload()

    def with_role(self, role: str) -> Self:
        self._payload.role = role
        return self

    def inactive(self) -> Self:
        self._payload.is_active = False
        return self

    def build(self) -> dict:
        # Возвращаем готовый dict для httpx/requests
        return self._payload.model_dump()
```

В тесте это читается как стихотворение (Fluent Interface):
```python
def test_admin_can_access_dashboard(api_client):
    # Создаем админа, остальные поля сгенерируются сами
    payload = UserPayloadBuilder().with_role("admin").build()
    
    response = api_client.post("/users", json=payload)
    assert response.status_code == 201
```

*🔥 Тренды 2025/2026:* На многих проектах ручные Билдеры заменяют библиотеками вроде `polyfactory` или `factory_boy` в связке с `Faker`, которые умеют генерировать Pydantic и SQLAlchemy модели «из коробки» на основе их типов.

---

## 2. Factory / Factory Method (Фабрика): Кроссплатформенность и выбор драйвера

**Проблема:** Запуск тестов на разных окружениях. Сегодня мы гоним тесты в Chrome, завтра в Firefox, а послезавтра — это мобильный тест на Android в Appium.

**Решение:** Паттерн Factory. Тестовый код вообще не должен знать, КАКОЙ браузер или клиент он использует. Он просит Фабрику: *"Дай мне браузер для текущего прогона"*, и Фабрика сама принимает решение на основе переменных окружения.

### 💻 Пример: WebDriver / Playwright Factory

```python
import os
from playwright.sync_api import sync_playwright, Browser

class BrowserFactory:
    @staticmethod
    def get_browser() -> Browser:
        # Читаем переменную окружения
        browser_type = os.getenv("BROWSER", "chrome").lower()
        playwright = sync_playwright().start()

        if browser_type == "chrome":
            return playwright.chromium.launch(headless=True)
        elif browser_type == "firefox":
            return playwright.firefox.launch(headless=True)
        elif browser_type == "webkit":
            return playwright.webkit.launch(headless=True)
        else:
            raise ValueError(f"Unsupported browser: {browser_type}")

# В фикстуре PyTest мы просто вызываем фабрику
# @pytest.fixture
# def browser_instance():
#     yield BrowserFactory.get_browser()
```

---

## 3. Singleton (Одиночка): Пул соединений и Конфигурация

**Концепция:** Singleton гарантирует, что у класса есть только *один* экземпляр, и предоставляет к нему глобальную точку доступа. 

В автотестах классически используется для:
* Менеджера конфигураций (чтение `config.yaml` один раз).
* Подключения к Базе Данных (чтобы не открывать 100 коннектов к БД на 100 тестов, а переиспользовать пул).

### ⚠️ Архитектурный нюанс SDET (Очень важно для собеса!)
В классическом ООП (Java/C++) Singleton реализуется через переопределение метода `__new__`. **НО! В Python и PyTest это считается антипаттерном**, особенно при запуске тестов в несколько потоков/процессов (через `pytest-xdist`). Если два воркера `xdist` попытаются обратиться к одному Singleton-объекту в памяти (например, socket соединения), всё сломается, так как процессы изолированы.

**Решение Python-way:** Использовать **Фикстуры PyTest с областью видимости `scope="session"`**. Это "естественный Singleton" в мире Pytest.

### 💻 Пример: Правильный "Singleton" для БД в реалиях PyTest

Вместо сложного класса `class Database(metaclass=SingletonMeta):`, мы делаем так в `conftest.py`:

```python
import pytest
from sqlalchemy import create_engine

# scope="session" гарантирует, что Engine будет создан ровно ОДИН раз
# для всей сессии тестирования (или один раз для каждого xdist-воркера)
@pytest.fixture(scope="session")
def db_engine():
    print("\n[INIT] Установка коннекта к БД...")
    engine = create_engine("postgresql+psycopg://user:pass@localhost:5432/mydb")
    
    yield engine  # Отдаем "синглтон" в тесты
    
    print("\n[TEARDOWN] Закрытие пула соединений...")
    engine.dispose()
```
*Этот подход надежнее, потокобезопаснее и не засоряет код метаклассами.*

---

## 🎤 Как это спросят на собеседовании

> **Вопрос 1: "Для чего используется паттерн Builder в автоматизации и чем он лучше использования словарей (`dict`) или kwargs?"**
* **Ответ Лида:** "Builder необходим для генерации сложных объектов (например, JSON-тела запроса или DTO). В отличие от `dict`, Builder предоставляет статическую типизацию, автокомплит в IDE и защищает от опечаток. Использование подходов типа Fluent Interface (`.with_role("admin").active()`) делает тестовый код самодокументируемым и легко читаемым. Для словарей мы этого лишены."

> **Вопрос 2: "Ты реализовал Singleton для подключения к БД (через `__new__`), но при запуске через `pytest -n 4` (pytest-xdist) скрипт начал падать с ошибками. Почему?"**
* **Ответ Лида:** "Потому что `pytest-xdist` создает независимые процессы (workers). У каждого процесса свое адресное пространство памяти. Настоящий глобальный Singleton (через память) между процессами в Python невозможен без использования менеджеров распределенной памяти (Redis, Memcached или `multiprocessing.Manager`). В мире Pytest функцию Singleton должен выполнять механизм Session Scope Fixtures, который безопасно поднимет по одному подключению для каждого воркера."

> **Вопрос 3: "У нас есть 5 микросервисов, для каждого написан свой API-клиент. В тестах постоянно приходится инициализировать их вручную. Какой паттерн применить?"**
* **Ответ Лида:** "Здесь идеально подойдет комбинация **Facade (Фасад)** и **Factory (Фабрика)**. Мы можем создать класс `ApiClientFactory` или общий `SystemFacade`, который скроет логику инстанцирования всех 5 клиентов. В тест мы будем передавать только один готовый объект фасада, из которого можно вызывать любой нужный сервис: `facade.billing_api.get_balance()`."
