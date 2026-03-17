# 📘 Глава 3. Проектирование и Design Patterns
## Тема 3.2. Паттерны UI-тестирования: Развитие Page Object / Page Factory, переход к Screenplay Pattern

### Введение: Почему паттерны эволюционируют?
По мере роста тестового покрытия в крупных компаниях (от 100 до 10 000 тестов), команды автоматизаторов сталкиваются с проблемой: классы страниц становятся огромными ("Божественные объекты"), код дублируется, а изменение одного UI-элемента ломает десятки не связанных напрямую тестов [1][4]. Согласно архитектурным принципам (SOLID), тестируемая система должна развиваться от монолитных страниц к компонентным абстракциям и поведенческим моделям [5].

---

### Часть 1. Page Factory: Наследие прошлого (Устарело в 2026)

**Page Factory** — это расширение классического POM, изначально популяризованное в экосистеме Java (Selenium WebDriver). Оно использовало аннотации (например, `@FindBy`) для декларативного поиска элементов и механизм ленивой инициализации (Lazy Initialization).

#### Почему от Page Factory отказались в современном стеке?
В 2026 году доминируют инструменты нового поколения: **Playwright** и **Cypress**. 
*   **Динамический DOM:** Современные фреймворки (React, Vue) постоянно перерисовывают элементы. Классический Page Factory часто приводил к ошибкам `StaleElementReferenceException`, так как кэшировал ссылки на уничтоженные элементы [3].
*   **Встроенное ожидание (Auto-waiting):** Playwright сам лениво ищет элемент (через `Locator`) строго в момент действия (клик, ввод текста) [3]. Необходимость в сложных фабриках, прокси-объектах и аннотациях `@FindBy` полностью отпала. 
*   **Вердикт 2026:** Page Factory считается антипаттерном в современных TS/Python фреймворках [5].

---

### Часть 2. Page Object Model (POM) и его эволюция

**Page Object Model (POM)** — это стандарт индустрии, который инкапсулирует локаторы и действия конкретной веб-страницы в один класс [1][4].

#### Проблемы классического POM на масштабе (Enterprise Scale):
1.  **Нарушение SRP (Single Responsibility):** Класс страницы часто содержит и структуру (локаторы), и бизнес-логику (методы вроде `login()`), и даже проверки [1].
2.  **Дублирование кода:** Если у вас есть сквозной компонент (например, шапка сайта или меню навигации), который присутствует на 50 страницах, классический POM заставляет либо дублировать его локаторы, либо выстраивать сложную и хрупкую иерархию наследования [5][6].

#### Эволюция: Page Component Model (PCM) / Flow Model
Чтобы решить проблемы POM, Senior-инженеры перешли к **компонентному подходу**. Вместо того чтобы описывать страницу целиком, мы описываем независимые виджеты (Компоненты) [4][6].
Например: `HeaderComponent`, `DatepickerWidget`. Страница (Page) теперь не содержит локаторов, а лишь выступает контейнером, который собирает эти компоненты воедино (Композиция вместо Наследования) [6].

---

### Часть 3. Screenplay Pattern (Стандарт BigTech 2026 года)

Когда POM и Component Model перестают справляться с переиспользованием бизнес-шагов между разными проектами и командами, архитекторы внедряют **Screenplay Pattern** (также известный как *Journey Pattern*) [2][4].

Screenplay — это сдвиг парадигмы. В отличие от POM, который фокусируется на *страницах* (UI-ориентированность), Screenplay фокусируется на **Пользователе и его поведении (Actor-centric)** [2][4]. Он идеально реализует принципы SOLID (особенно Open/Closed и Single Responsibility)[2].

#### Пять столпов Screenplay (Терминология):
1.  **Actor (Актер):** Кто выполняет тест? (Например, "Администратор", "Гость"). У актера есть *Способности*.
2.  **Ability (Способность):** Что актер может делать? (Например, `BrowseTheWeb` через Playwright, `CallAnApi` через httpx) [4].
3.  **Task (Задача):** Высокоуровневый бизнес-шаг. Состоит из мелких действий. (Например, `Login.with_credentials(user, pass)`).
4.  **Interaction (Взаимодействие):** Низкоуровневое действие с UI или API. (Например, `Click.on(LOGIN_BUTTON)`, `Enter.text("user").into(USERNAME_FIELD)`) [4].
5.  **Question (Вопрос):** Запрос состояния системы для последующего Assert'а. (Например, `Text.of(WELCOME_MESSAGE)`).

#### Пример архитектуры Screenplay (Python + Playwright + библиотека ScreenPy [7]):

```python
# 1. Локаторы (Target) хранятся отдельно от логики
from screenpy_playwright import Target
USERNAME_FIELD = Target.the("Username field").located_by("#user")
LOGIN_BUTTON = Target.the("Login button").located_by("#submit")

# 2. Задача (Task) комбинирует низкоуровневые взаимодействия
from screenpy import Actor, Task
from screenpy_playwright import Enter, Click

class Login(Task):
    def __init__(self, username, password):
        self.username = username
        self.password = password

    def perform_as(self, actor: Actor):
        actor.attempts_to(
            Enter.the_text(self.username).into(USERNAME_FIELD),
            Enter.the_text(self.password).into(PASSWORD_FIELD),
            Click.on(LOGIN_BUTTON)
        )

# 3. Сам автотест читается как естественный язык (BDD-стиль)
from screenpy import Actor
from screenpy_playwright import BrowseTheWeb

def test_admin_can_login(playwright_page):
    # Инициализация актера и его способностей
    admin = Actor.named("Alice").who_can(BrowseTheWeb.using(playwright_page))
    
    # Выполнение бизнес-задачи
    admin.attempts_to(
        Login("admin_user", "secure_pass123")
    )
    
    # Проверка через Question
    admin.should(
        See.the(Text.of_the(WELCOME_MESSAGE), ReadsExactly("Welcome, Alice!"))
    )
```

---

### Часть 4. Best Practices & Bad Practices (Что выбрать?)

Внедрение Screenplay Pattern не является серебряной пулей; оно имеет высокий порог входа [4]. Senior SDET должен уметь обосновать выбор паттерна бизнесу.

#### ❌ Bad Practices
1.  **Внедрение Screenplay в маленькие проекты:** Использование Screenplay для проекта из 50 тестов и 2-х страниц приведет к чрезмерному усложнению (Overengineering). Для таких задач обычного POM более чем достаточно [1].
2.  **POM с глубоким наследованием:** Создание архитектуры вида `BasePage -> AuthenticatedPage -> DashboardPage -> AdminDashboardPage`. Наследование ломает гибкость. В 2026 году стандартом является Композиция (Composition over Inheritance) [3].

#### ✅ Best Practices (Архитектурные рекомендации)
1.  **POM для старта, Screenplay для масштабирования:** Если команда быстро растет (более 5 автоматизаторов) и тестирует экосистему из множества приложений (например, 60+ микро-фронтендов), Screenplay позволит создать единую библиотеку Задач (Tasks) и Взаимодействий (Interactions), доступную всем командам без дублирования кода [1][2].
2.  **Поведенческий фокус (Behavior-Driven):** Screenplay заставляет программистов думать не о структуре страницы, а о поведении пользователя, что снижает когнитивную нагрузку при чтении упавших логов [3]. При падении отчета вы видите не `LoginPage.click_button() failed`, а `Alice failed to Login because the Login Button was hidden`.

---

### 📚 Sources (Источники)
1. [Page Object vs Screenplay Pattern — Which One Scales Better for Large Teams? (Medium, Nov 2025)](https://medium.com/@gunashekar.qa/page-object-vs-screenplay-pattern-which-one-scales-better-for-large-teams-9d7a26f8b54b)
2. [Screen Play vs Page Object pattern (Software Quality Assurance Stack Exchange)](https://sqa.stackexchange.com/questions/27641/screen-play-vs-page-object-pattern)
3. [Screenplay vs. Page Objects (Boa Constrictor - GitHub Pages)](https://boaconstrictor.github.io/Boa.Constrictor/docs/screenplay-vs-page-objects)
4. [Test Automation Design Pattern: POM, Flow Model, Screenplay (Cloudflight Engineering, Dec 2025)](https://www.cloudflight.io/en/blog/test-automation-design-pattern-pom-flow-model-screenplay/)
5. [Beyond the Page Object Model: Unlocking the Screenplay Pattern (ERNI)](https://betterask.erni/wp-content/uploads/2024/09/Beyond-the-Page-Object-Model-Unlocking-the-Screenplay-Pattern.pdf)
6. [Revolution or Evolution of Page Object Model? (Medium)](https://medium.com/@artem.sokovets/revolution-or-evolution-of-page-object-model-7142d766cf1c)
7. [ScreenPy: Screenplay Pattern for Python! (Official Documentation)](https://screenpy-docs.readthedocs.io/en/latest/)
