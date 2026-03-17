# 📘 Глава 3. Проектирование и Design Patterns
## Тема 3.3. Структурные и порождающие паттерны: Singleton, Factory, Dependency Injection

### Введение: Уровень Enterprise-проектирования
Когда автоматизатор перерастает позицию Middle, он понимает, что фреймворк — это не просто набор тестов, а сложная система, управляющая ресурсами (браузерами, базами данных, логгерами). 

В 2026 году стандартом индустрии является непрерывное тестирование (Continuous Testing) с параллельным запуском тысяч тестов в облачных инфраструктурах. Если не использовать паттерны проектирования, управление ресурсами превратится в хаос, а тесты станут нестабильными (flaky).

### Паттерн 1. Singleton (Одиночка): От классики к Антипаттерну 2026 года

**Суть паттерна:** Гарантирует, что у класса есть только один экземпляр во всем приложении, и предоставляет глобальную точку доступа к нему [2][5].

#### ❌ Bad Practice: Singleton для WebDriver
Исторически (в эпоху последовательного запуска тестов) QA-инженеры обожали делать WebDriver синглтоном, чтобы легко вызывать `Driver.get_instance()` из любого места в коде. 
В реалиях 2026 года использование Singleton для WebDriver (или Playwright Page) официально признано **Антипаттерном** [1][2][5]. 
*Почему?* Главное требование современного CI/CD — **параллельное выполнение** (Parallel Execution). Если ваши тесты запускаются в 10 потоков (через `pytest-xdist`), и все они делят один Singleton-браузер, они начнут перехватывать управление друг у друга, кликать не туда и блокировать потоки (Race conditions) [1][3].

#### ✅ Best Practice 1: Thread-Local Singleton
Если вам жестко необходимо глобальное управление драйвером в многопоточной среде, Senior SDET использует паттерн **Thread-Local Singleton** [3]. Он гарантирует, что внутри *каждого отдельного потока* будет создана ровно одна инстанция драйвера, изолированная от остальных потоков [2][3].

#### ✅ Best Practice 2: Истинное назначение Singleton
В правильном фреймворке Singleton используется только для **неизменяемых (Immutable) ресурсов**, которые безопасны для потоков (Thread-safe) [4]:
*   Менеджеры конфигураций (чтение `config.yaml` или `env` переменных один раз при старте).
*   Пул подключений к базе данных (DB Connection Pool).
*   Системные Логгеры (Global Logger).

---

### Паттерн 2. Factory Pattern (Фабрика): Управление зоопарком браузеров

**Суть паттерна:** Скрывает логику создания объектов от пользователя. Клиент просит у Фабрики объект с заданными параметрами, и Фабрика сама решает, какой именно класс инстанцировать[6][7].

#### Зачем это SDET?
Тесты не должны знать, *как* запускается браузер. Они просто хотят получить готовый `driver` или `page` [7]. Сегодня тесты запускаются локально в Chrome, завтра — на удаленном Selenium Grid / Moon Hub, послезавтра — в режиме эмуляции iPhone 15 Pro.

#### ✅ Best Practice: WebDriver / Playwright Factory
Вместо хардкода (жесткого прописывания логики в тесте), мы создаем фабричный класс, который принимает переменные окружения и отдает нужный инстанс браузера [6][7].

```python
# ПРИМЕР: Упрощенная Фабрика драйверов (Python)
from selenium import webdriver

class DriverFactory:
    @staticmethod
    def get_driver(browser_name: str, environment: str) -> webdriver.Remote:
        if environment == "local":
            if browser_name == "chrome":
                options = webdriver.ChromeOptions()
                options.add_argument("--headless")
                return webdriver.Chrome(options=options)
            elif browser_name == "firefox":
                return webdriver.Firefox()
        elif environment == "remote":
            # Возвращаем драйвер для облачной фермы (например, Selenoid/Moon)
            options = webdriver.ChromeOptions()
            return webdriver.Remote(command_executor="http://hub:4444/wd/hub", options=options)
            
        raise ValueError(f"Unsupported browser/env: {browser_name} / {environment}")
```
*В тестах логика сводится к минимуму: `driver = DriverFactory.get_driver(config.browser, config.env)`. Это полностью соответствует принципу Open/Closed (SOLID) — для добавления нового браузера мы не трогаем тесты, а лишь расширяем Фабрику[6].*

---

### Паттерн 3. Dependency Injection (Внедрение зависимостей)

**Суть паттерна:** Класс не должен сам создавать свои зависимости, они должны передаваться ему извне (инжектироваться) при создании. Это главное правило инверсии зависимостей (DIP из SOLID).

#### ❌ Bad Practice (Скрытые зависимости)
Если класс Page Object или класс Теста сам инстанцирует свой браузер или API-клиент, такой код невозможно изолированно протестировать, мокировать или распараллелить.
```python
# ПЛОХО: Page Object сам создает драйвер. 
class LoginPage:
    def __init__(self):
        self.driver = webdriver.Chrome() # Нарушение DI
```

#### ✅ Best Practice: Pytest как DI-контейнер
В мире Java для DI используют тяжелые фреймворки (Spring, Guice). В экосистеме Python (в 2026 году) **фреймворк Pytest сам по себе является мощнейшим контейнером внедрения зависимостей** [8][9]! 
Фикстуры (Fixtures) в Pytest построены на паттерне Dependency Injection. Вы объявляете, что тесту нужен драйвер, и Pytest автоматически "инжектирует" (внедряет) его в аргументы функции [8][10].

```python
import pytest
from core.factory import DriverFactory
from pages.login_page import LoginPage

# 1. Фикстура выступает провайдером зависимости (используя Фабрику)
@pytest.fixture(scope="function")
def driver(request):
    browser_type = request.config.getoption("--browser")
    driver_instance = DriverFactory.get_driver(browser_type, "local")
    yield driver_instance
    driver_instance.quit() # Безопасный Teardown

# 2. Фикстура инжектирует драйвер в Page Object
@pytest.fixture
def login_page(driver):
    return LoginPage(driver) # Внедрение зависимости через конструктор (Constructor Injection)

# 3. Тест просто запрашивает нужную зависимость (Pytest делает остальное)
def test_user_can_login(login_page):
    login_page.login("admin", "password123")
    assert login_page.is_dashboard_visible()
```

### Резюме архитектора
В 2026 году Senior SDET должен помнить следующее правило: **Singleton** используем с осторожностью (только для неизменяемых конфигураций/логгеров), чтобы не сломать параллелизацию [1]. Для создания сложных объектов (браузеры, API-клиенты с разными заголовками) используем **Factory Pattern**[6]. А для связывания всего этого в единый фреймворк и доставки ресурсов в тесты используем **Dependency Injection** через нативные механизмы `pytest` (Фикстуры) [8].

***

### 📚 Sources (Источники)
1. [Best Practices for Designing a Test Automation Framework | Medium (Sep 2024)](https://medium.com/@rajuppadhyay/best-practices-for-designing-a-test-automation-framework-9a5624773c82)
2. [Singleton Design Pattern: How to Use It In Test Automation | Testomat.io (Nov 2023)](https://testomat.io/blog/singleton-design-pattern-how-to-use-it-in-test-automation/)
3. [One Driver to Rule Them All: Thread-Local Singleton Pattern | Medium (Apr 2025)](https://medium.com/@meenaakannan/one-driver-to-rule-them-all-understanding-the-thread-local-singleton-pattern-in-selenium-c0e81c1c9c45)
4. [Leveraging Design Patterns in Automation Testing: A Paradigm Shift | SAPUB (Aug 2024)](http://article.sapub.org/10.5923.j.se.20241401.01.html)
5. [Why is Singleton considered an anti-pattern? | Stack Overflow](https://stackoverflow.com/questions/137975/why-is-singleton-considered-an-anti-pattern)
6. [Design Patterns in Test Automation (Factory Pattern) | Medium (Apr 2025)](https://medium.com/@elenamarin09/design-patterns-in-test-automation-90226c6d0426)
7. [Selenium WebDriver - Design Patterns in Test Automation - Factory Pattern | Vinsguru (Mar 2017/Updated)](https://www.vinsguru.com/selenium-webdriver-design-patterns-in-test-automation-factory-pattern/)
8. [How to Use PyTest Framework to Improve Selenium | Perforce BlazeMeter](https://www.blazemeter.com/blog/pytest-framework-improve-selenium)
9. [pytest Tutorial: Effective Python Testing | RealPython](https://realpython.com/pytest-python-testing/)
10. [Pytest-BDD: dependency injection via fixtures | ReadTheDocs (v8.1.0)](https://pytest-bdd.readthedocs.io/en/latest/)
