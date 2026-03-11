# Блок 2: Инженерия и написание тестов (База)
## Тема 4: Базовая архитектура и паттерны (Как не писать спагетти-код)

Когда ручной тестировщик только начинает изучать автоматизацию, первое желание — написать код в виде сплошной простыни текста: *«найди поле -> введи логин -> найди кнопку -> кликни -> проверь текст»*. 

Такой подход приводит к созданию **«спагетти-кода»** — запутанного, хрупкого и нечитаемого монолита. Если у вас 50 таких тестов, и разработчик изменит ID кнопки входа, вам придется вручную исправлять код в 50 разных местах [[1](https://www.virtuosoqa.com/post/page-object-model-selenium-cypress)]. Чтобы этого избежать, инженеры используют архитектурные паттерны.

В этой теме мы разберем базовые правила организации кода, которые в 2026 году ожидают от каждого Junior AQA на техническом собеседовании.

---

### 1. Базовые принципы чистого кода (Clean Code)

Вам не нужно быть Senior-разработчиком, чтобы писать хорошие тесты, но ваш код должен подчиняться трем фундаментальным правилам программирования [[2](https://medium.com/@jayeshsanghani88/5-powerful-rules-every-developer-should-follow-to-write-clean-code-5b2c6e5df24f)][[3](https://www.pullchecklist.com/posts/clean-coding-principles)].

#### DRY (Don't Repeat Yourself — Не повторяйся)
Самое важное правило автоматизатора. Если вы ловите себя на том, что копируете один и тот же кусок кода (например, шаги авторизации) из одного теста в другой — вы нарушаете DRY [[2](https://medium.com/@jayeshsanghani88/5-powerful-rules-every-developer-should-follow-to-write-clean-code-5b2c6e5df24f)]. 
* **Как правильно:** Вынесите этот код в отдельную функцию (например, `login()`) и просто вызывайте её в нужных тестах. Если логика авторизации изменится, вы исправите её только в одном месте, и все тесты продолжат работать [[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)].

#### KISS (Keep It Simple, Stupid — Делай проще)
Тесты должны быть максимально простыми и прямолинейными. 
* **Как правильно:** Избегайте сложной логики (вложенных циклов `for`, сложных конструкций `if/else`) внутри самого автотеста [[2](https://medium.com/@jayeshsanghani88/5-powerful-rules-every-developer-should-follow-to-write-clean-code-5b2c6e5df24f)][[3](https://www.pullchecklist.com/posts/clean-coding-principles)]. Автотест не должен «думать», он должен выполнять четкие шаги. Если тест сложен для понимания с первого взгляда — он написан плохо.

#### SoC (Separation of Concerns — Разделение ответственности)
Каждый модуль, класс или функция должны отвечать только за одну конкретную задачу [[2](https://medium.com/@jayeshsanghani88/5-powerful-rules-every-developer-should-follow-to-write-clean-code-5b2c6e5df24f)]. Не смешивайте в одном месте поиск элементов на странице, действия пользователя и проверки результата [[5](https://www.testmuai.com/learning-hub/playwright-page-object-model/)].

---

### 2. Page Object Model (POM) — Золотой стандарт индустрии

**Page Object Model (POM)** — это архитектурный паттерн (шаблон проектирования), который реализует принцип *Separation of Concerns* в UI-тестировании [[1](https://www.virtuosoqa.com/post/page-object-model-selenium-cypress)]. Несмотря на появление новых фреймворков и AI-инструментов, в 2026 году POM остается фундаментом для Playwright, Cypress и Selenium [[1](https://www.virtuosoqa.com/post/page-object-model-selenium-cypress)][[6](https://www.skyvern.com/blog/page-object-model-guide/)].

**Суть паттерна:** 
Каждая веб-страница (или ее крупный компонент, например, шапка сайта) описывается в виде отдельного Класса (файла). Этот класс является «словарем» и «инструкцией» к странице [[6](https://www.skyvern.com/blog/page-object-model-guide/)][[7](https://www.browserstack.com/guide/page-object-model-with-playwright)].

В правильном Page Object хранятся только две вещи:
1. **Локаторы (Locators):** «Где находится элемент?» (кнопки, поля ввода, чекбоксы).
2. **Методы (Actions):** «Что мы можем сделать на этой странице?» (заполнить форму, нажать кнопку, прочитать текст) [[1](https://www.virtuosoqa.com/post/page-object-model-selenium-cypress)][[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)].

#### ❌ Как выглядит спагетти-код (Без POM):
Локаторы зашиты прямо (захардкожены) в тело теста.
```python
async def test_user_can_login(page: Page):
    # Тест сам ищет элементы и сам совершает действия. Это плохо!
    await page.locator('#email-input').fill('user@test.com')
    await page.locator('#password-input').fill('123456')
    await page.locator('.btn-login-submit').click()
    
    await expect(page.locator('.welcome-message')).to_be_visible()
```
*Проблема:* Если класс кнопки `.btn-login-submit` изменится на `.btn-primary`, вам придется искать и менять этот селектор во всех существующих тестах [[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)].

#### ✅ Как выглядит архитектура с POM:
Тест не знает *КАК* устроена страница, он просто отдает команды Page Object'у [[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)].
```python
# Файл 1: login_page.py (Сам Page Object)
class LoginPage:
    def __init__(self, page: Page):
        self.page = page
        self.email_input = page.locator('#email-input')  # Локатор хранится здесь
        self.password_input = page.locator('#password-input')
        self.login_button = page.locator('.btn-login-submit')

    # Метод, объединяющий действия
    def login(self, email: str, password: str) -> None:
        self.email_input.fill(email)
        self.password_input.fill(password)
        self.login_button.click()

# Файл 2: test_login.py (Сам Автотест)
def test_user_can_login(page: Page):
    login_page = LoginPage(page)  # Подключаем страницу
    
    # Тест читается как простой текст!
    login_page.login('user@test.com', '123456') 
    
    expect(page.locator('.welcome-message')).to_be_visible()
```
*Решение:* Если кнопка изменилась, мы меняем локатор *только один раз* внутри файла `login_page.py` [[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)][[5](https://www.testmuai.com/learning-hub/playwright-page-object-model/)].

---

### 3. Главное правило: Разделяй Шаги и Проверки (Asserts)

Частая ошибка Junior-инженеров — помещать проверки (Asserts, ожидаемые результаты) прямо внутрь Page Object [[1](https://www.virtuosoqa.com/post/page-object-model-selenium-cypress)]. 

**Почему это антипаттерн в 2026 году:**
Page Object должен только *предоставлять информацию* или *выполнять действия*, но не принимать решения о том, пройден тест или нет [[1](https://www.virtuosoqa.com/post/page-object-model-selenium-cypress)][[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)]. 

* **Плохо (Assert внутри POM):** В методе `login()` вы пишете проверку `expect(header).toBeVisible()`. А что, если в другом тесте вы хотите проверить сценарий *неудачной* авторизации с неверным паролем? Ваш метод упадет, так как будет жестко ожидать успешного входа.
* **Хорошо:** Page Object выполняет действие `login()`, а сам `Assert` находится в файле теста (test). Тест сам решает, что именно он хочет проверить: появление ошибки или успешный вход [[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)].

> **Исключение:** В современных фреймворках (например, Playwright) допускается создавать специальные методы-проверки в POM только если это сильно упрощает чтение, но классическое правило гласит: **Actions in POM, Assertions in Tests** (Действия в POM, Проверки в Тестах) [[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)].

---

### Резюме для самопроверки

1. **DRY (Не повторяйся)**: Никогда не дублируйте один и тот же код в разных тестах. Выносите повторяющиеся шаги в переиспользуемые функции (методы) [[2](https://medium.com/@jayeshsanghani88/5-powerful-rules-every-developer-should-follow-to-write-clean-code-5b2c6e5df24f)][[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)].
2. **KISS (Делай проще)**: Автотесты не должны содержать сложной логики ветвления. Тест должен быть линейным и легко читаемым [[2](https://medium.com/@jayeshsanghani88/5-powerful-rules-every-developer-should-follow-to-write-clean-code-5b2c6e5df24f)][[3](https://www.pullchecklist.com/posts/clean-coding-principles)].
3. **Page Object Model (POM)**: Это паттерн, который отделяет структуру страницы (локаторы) от бизнес-логики (самих тестов) [[1](https://www.virtuosoqa.com/post/page-object-model-selenium-cypress)].
4. **Хранение локаторов**: Все локаторы и селекторы должны жить в файлах страниц (Page Objects), а не в файлах тестов [[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)].
5. **Разделение ответственности**: Page Object кликает и вводит текст. Файл теста — организует процесс и проводит финальную проверку (Assert) [[4](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)][[5](https://www.testmuai.com/learning-hub/playwright-page-object-model/)].

Поняв эту концепцию, вы навсегда избавите себя от написания хрупкого, постоянно ломающегося спагетти-кода. В следующей теме мы разберем не менее важный фундамент стабильности — **Работу с тестовыми данными** (почему нельзя хардкодить пользователей и как готовить данные правильно).

Sources
1. [Page Object Model in Selenium & Cypress: Complete Guide](https://www.virtuosoqa.com/post/page-object-model-selenium-cypress)
2. [5 Powerful Rules Every Developer Should Follow to Write Clean Code](https://medium.com/@jayeshsanghani88/5-powerful-rules-every-developer-should-follow-to-write-clean-code-5b2c6e5df24f)
3. [7 Clean Coding Principles Every Developer Should Know](https://www.pullchecklist.com/posts/clean-coding-principles)
4. [Page Object Model Playwright (2026): Best TypeScript Guide](https://www.anton.qa/blog/posts/page-object-model-in-playwright-with-typescript-complete-guide)
5. [Playwright Page Object Model : A Complete Guide](https://www.testmuai.com/learning-hub/playwright-page-object-model/)
6. [Mastering Page Object Model (POM) in Test Automation - October 2025 Edition](https://www.skyvern.com/blog/page-object-model-guide/)
7. [Page Object Model with Playwright: Tutorial [2026]](https://www.browserstack.com/guide/page-object-model-with-playwright)
