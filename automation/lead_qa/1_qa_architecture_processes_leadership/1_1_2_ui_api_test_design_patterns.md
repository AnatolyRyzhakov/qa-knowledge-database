# 📂 Module 1.1.2. Паттерны проектирования UI/API тестов (Page Object, Screenplay, Facade)

Привет! Продолжаем проектировать архитектуру. Когда ты только начинаешь автоматизацию, писать всё подряд в одном скрипте (Procedural Programming) — это быстро и весело. Но когда тестов становится 500+, а в команду приходят новые люди, такой код превращается в «спагетти», которое невозможно поддерживать.

На позиции Lead SDET от тебя ждут внедрения паттернов, которые сделают фреймворк соответствующим принципам **SOLID, DRY (Don't Repeat Yourself)** и **KISS**. В 2025–2026 годах подходы к паттернам сильно эволюционировали. Давай разберем три главных столпа архитектуры автотестов на сегодняшний день.

---

## 1. Page Object Model (POM): Компонентный подход (Тренды 2025)

**Page Object** — это нестареющая классика. Суть: каждая страница веб-приложения (или её часть) описывается отдельным классом, который хранит локаторы и методы взаимодействия.

**Главная проблема классического POM:** Со временем классы страниц раздуваются до тысяч строк кода (антипаттерн "God Object"). Если на 10 страницах есть одинаковый Header или таблица, локаторы начинают дублироваться.

**Решение 2025 года: Component-based Page Object.**
Мы больше не пишем монолитные классы страниц. Мы собираем страницы из независимых компонентов (Composition over Inheritance).

**Золотое правило POM (Best Practice Playwright):** Page Object **НЕ ДОЛЖЕН** содержать проверок (`assert` / `expect`). Страница только возвращает состояние или локатор, а сам `assert` происходит в тестовом файле.

### 💻 Пример (Python + Playwright + ООП)

```python
from playwright.sync_api import Page, Locator
from typing import Self

# 1. Выделяем переиспользуемый компонент
class HeaderComponent:
    def __init__(self, page: Page):
        self.page = page
        self.logout_button: Locator = page.locator("#logout")
        self.profile_icon: Locator = page.locator(".profile-icon")

    def click_logout(self) -> None:
        self.logout_button.click()

# 2. Собираем страницу с использованием Компонента
class DashboardPage:
    def __init__(self, page: Page):
        self.page = page
        self.url = "/dashboard"
        # Композиция: подключаем компонент Header
        self.header = HeaderComponent(page)
        self.welcome_message: Locator = page.locator("h1.welcome")

    def navigate(self) -> Self:
        self.page.goto(self.url)
        return self

    def get_welcome_text(self) -> str:
        # Дожидаемся видимости и возвращаем текст (никаких assert здесь!)
        return self.welcome_message.inner_text()
```

В самом тесте (`test_dashboard.py`):
```python
from playwright.sync_api import expect

def test_user_can_logout(page):
    dashboard = DashboardPage(page).navigate()
    
    # Assert находится в тесте, а не внутри Page Object
    expect(dashboard.welcome_message).to_be_visible()
    
    # Взаимодействие через компонент
    dashboard.header.click_logout()
```

---

## 2. Screenplay Pattern (Поведенческий паттерн)

В сложных Enterprise-проектах POM перестает справляться. На смену ему приходит **Screenplay Pattern**. Это подход, основанный на принципах BDD, где в центре архитектуры находится не Страница, а **Пользователь (Actor)**.

Screenplay разделяет ответственность (Single Responsibility Principle) на 4 сущности:
1. **Actor (Актер):** Тот, кто выполняет действия (Например, `Admin`, `Guest`).
2. **Abilities (Способности):** То, что умеет Актер (Например, `BrowseTheWeb`, `CallAnApi`).
3. **Tasks & Interactions (Задачи и Взаимодействия):** Высокоуровневые бизнес-действия (`LoginToSystem`) и низкоуровневые шаги (`Click.on(Button)`).
4. **Questions (Вопросы):** Получение информации из системы (`TheTextOf(WelcomeMessage)`).

В экосистеме Python стандартом де-факто для этого паттерна стала библиотека **ScreenPy** (которая в 2025/2026 году получила активное развитие).

### 💻 Пример (ScreenPy Style)

```python
from screenpy import Actor
from screenpy_playwright.abilities import BrowseTheWeb
from screenpy.actions import Click, Enter
from screenpy_playwright.questions import Text

# Инициализация актера со способностью управлять браузером
luke = Actor.named("Luke").who_can(BrowseTheWeb.using_playwright(page))

# Выполнение высокоуровневых действий (читается как обычный английский текст)
luke.attempts_to(
    Enter.the_text("luke@jedi.com").into(LOGIN_FIELD),
    Enter.the_text("secret").into(PASSWORD_FIELD),
    Click.on(SUBMIT_BUTTON)
)

# Проверка (Question)
assert luke.asks_for(Text.of(WELCOME_MESSAGE)) == "Welcome, Luke!"
```

**Почему это круто?** Бизнес-логика (`Enter`, `Click`, `LoginTask`) переиспользуется на 100%. Мы избавляемся от дублирования кода между десятками Page-классов.

---

## 3. Facade Pattern (Фасад для API и сложной логики)

Паттерн **Facade (Фасад)** скрывает сложную внутреннюю структуру системы за простым и понятным интерфейсом.

Это **Must Have** для интеграционных и API автотестов. Представь, что для создания пользователя со статусом "Active" в микросервисной архитектуре нужно дернуть 3 разных сервиса (Auth API, User API, Billing API) и базу данных. Если писать это в тесте напрямую, тест станет огромным и нечитаемым.

### 💻 Пример (Паттерн Facade в API тестах)

```python
import httpx

# Низкоуровневые API клиенты микросервисов
class AuthApiClient:
    def get_token(self) -> str: ...

class UserApiClient:
    def create_user(self, token: str, data: dict) -> str: ...

class BillingApiClient:
    def set_balance(self, user_id: str, amount: int) -> None: ...

# ФАСАД: скрывает сложность интеграций
class SystemFacade:
    def __init__(self):
        self.auth = AuthApiClient()
        self.user = UserApiClient()
        self.billing = BillingApiClient()

    # ОДИН простой метод для тестов, который делает всё "под капотом"
    def setup_active_user_with_balance(self, email: str, balance: int) -> str:
        token = self.auth.get_token()
        user_id = self.user.create_user(token, {"email": email, "status": "active"})
        self.billing.set_balance(user_id, balance)
        return user_id
```

В тесте это выглядит максимально элегантно:
```python
def test_user_can_buy_item(system_facade: SystemFacade):
    # Тесту не нужно знать про токены и 3 разных микросервиса. Фасад делает "магию".
    user_id = system_facade.setup_active_user_with_balance("test@mail.com", balance=100)
    # ... дальнейшие действия теста ...
```

---

## 🎤 Как это спросят на собеседовании

> **Вопрос 1: "Где должны храниться `assert` (проверки) при использовании паттерна Page Object?"**
* **Ответ Лида:** "Проверки строго должны находиться в самих тестах (тестовых методах). Page Object должен отвечать только за инкапсуляцию логики работы со страницей: клики, ввод текста, чтение значений. Если добавить `assert` внутрь Page Object, мы нарушим принцип единой ответственности (SRP) — метод начнет и собирать данные, и принимать решения об успехе теста. Исключением может быть только метод типа `is_loaded()`, который проверяет, что страница действительно открылась."

> **Вопрос 2: "Твой POM разросся, появился класс на 2000 строк. Что будешь делать?"**
* **Ответ Лида:** "Я применю подход Composition over Inheritance (Композиция вместо наследования). Я разобью монолитный класс страницы на мелкие независимые Компоненты: `SidebarComponent`, `TableComponent`, `HeaderComponent`. Сама страница будет лишь контейнером для этих компонентов. Это уменьшит дублирование кода и повысит читаемость."

> **Вопрос 3: "В чём фундаментальная разница между Page Object и Screenplay?"**
* **Ответ Лида:** "POM строится вокруг **структуры UI** (страниц). Screenplay строится вокруг **Пользователя (Actor)** и его **бизнес-целей**. В POM мы вызываем методы у страниц (`LoginPage.login()`). В Screenplay мы даем "пользователю" задачу (`Actor.attempts_to(LoginTask())`). Screenplay лучше масштабируется на гигантских проектах и API-тестировании, так как заставляет писать максимально модульный код, но у него выше порог входа для новичков."

---
Sources:
1. [Enhancing the Page Object Pattern with Reusable Components in Playwright (Python)](https://python.plainenglish.io/enhancing-the-page-object-pattern-with-reusable-components-in-playwright-python-5a6d4481de30)
2. [Modern Web Test Automation with Playwright and Python](https://www.coursera.org/learn/packt-modern-web-test-automation-with-playwright-and-python-4dnk9)
3. [The Screenplay Pattern: Modern to Test Automation](https://wearecommunity.io/communities/testautomation/articles/7203)
4. [Implementing the Screenplay Pattern with Python-Part 1: the theory](https://medium.com/@bernd.bornhausen/implementing-the-screenplay-pattern-with-python-part-1-the-theory-89d56939d3d7)
5. [playwright_python_practice](https://github.com/Goraved/playwright_python_practice)
6. [I Built a Screenplay Pattern Framework for Python — Here’s Why](https://medium.com/@senior.fasihi/mar-21-2026-91ae404196b9)
7. [screenpy](https://pypi.org/project/screenpy/)
