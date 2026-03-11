# Блок 2: Инженерия и написание тестов (База)
## Тема 6: Обзор современных инструментов по языкам (Стеки 2026 года)

### Экосистема 1: TypeScript / JavaScript (Современный Web)
**Для кого:** Идеальный выбор, если вы хотите тестировать современные веб-приложения (SPA на React, Vue) и работать в тесной связке с Frontend-разработчиками. В 2026 году TypeScript является абсолютным лидером в Web-автоматизации [[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)][[4](https://www.testdevlab.com/blog/top-20-software-testing-automation-frameworks-for-web-and-mobile-in-2025)].

*   **UI-фреймворки (Битва титанов):**
    *   **Playwright:** Индустриальный стандарт 2026 года от Microsoft [[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)][[4](https://www.testdevlab.com/blog/top-20-software-testing-automation-frameworks-for-web-and-mobile-in-2025)]. Он невероятно быстрый, "из коробки" поддерживает параллельный запуск (что сокращает время прогона тестов в 2 раза) и умеет сам дожидаться загрузки элементов, избавляя вас от flaky tests [[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)]. 
    *   **Cypress:** Любимец фронтенд-разработчиков [[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)][[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)]. Он работает прямо внутри браузера, что дает уникальную возможность "путешествовать во времени" (Time-travel debugging) по шагам теста [[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)][[5](https://blog.greenroots.info/top-10-qa-automation-tools-for-startups-in-2026)]. *Минус:* поддерживает только JS/TS и имеет ограничения при работе с несколькими вкладками браузера.
*   **API-тестирование:** 
    В этом стеке AQA-инженеры стараются не плодить зоопарк инструментов. Запросы к бэкенду отправляются **встроенными методами** самих UI-фреймворков (например, `page.request` в Playwright или `cy.request()` в Cypress) [[5](https://blog.greenroots.info/top-10-qa-automation-tools-for-startups-in-2026)]. Если нужна отдельная библиотека, используют **Axios**.
*   **Тест-раннеры (Движки):** 
    Встроенные. Вам не нужно настраивать сторонние инструменты — и Playwright, и Cypress сами находят, запускают тесты и генерируют красивые отчеты [[6](https://medium.com/sdet-insights/comparing-uft-developer-vs-playwright-vs-selenium-for-enterprise-applications-in-2025-eab71ba8208a)].

---

### Экосистема 2: Python (Backend, Данные и Универсальность)
**Для кого:** Лучший выбор для новичка. У Python самый низкий порог входа — его синтаксис читается как простой английский текст [[4](https://www.testdevlab.com/blog/top-20-software-testing-automation-frameworks-for-web-and-mobile-in-2025)][[7](https://www.testdevlab.com/blog/reduce-time-and-effort-with-automated-testing)]. Этот стек доминирует в компаниях, тестирующих сложные бэкенд-системы, базы данных, AI-продукты или IoT (интернет вещей).

*   **Тест-раннер (Сердце экосистемы):**
    В мире Python всё вращается вокруг фреймворка **Pytest**. Это абсолютный лидер. Его главная суперсила — *фикстуры (fixtures)*. Это невероятно удобный механизм, который автоматически подготавливает тестовые данные (например, создает пользователя в БД) до начала теста и очищает их после.
*   **UI-фреймворки:**
    Еще пару лет назад здесь безоговорочно правил Selenium. Но в 2026 году стандартом стала связка **Playwright for Python** [[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)][[4](https://www.testdevlab.com/blog/top-20-software-testing-automation-frameworks-for-web-and-mobile-in-2025)]. Он позволяет писать современные, быстрые и надежные UI-тесты на любимом "питоне" [[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)][[6](https://medium.com/sdet-insights/comparing-uft-developer-vs-playwright-vs-selenium-for-enterprise-applications-in-2025-eab71ba8208a)].
*   **API-тестирование:**
    Здесь используется встроенная или сторонняя библиотека **Requests** (или её асинхронный аналог `httpx`). Написать API-тест на Python можно буквально в три строчки кода: `response = requests.get('url')`, а затем `assert response.status_code == 200`.

---

### Экосистема 3: Java / C# (Enterprise и Финтех)
**Для кого:** Это мир крупных корпораций, банков, биллинговых систем и высоких нагрузок. Языки со строгой типизацией (ООП) учат писать максимально надежный код. Если вы хотите работать в финтехе, от вас будут ждать знания именно этого стека [[1](https://luxequality.com/blog/automated-functional-testing/)][[6](https://medium.com/sdet-insights/comparing-uft-developer-vs-playwright-vs-selenium-for-enterprise-applications-in-2025-eab71ba8208a)].

*   **UI-фреймворки (Наследие и мощь):**
    Здесь продолжает доминировать **Selenium WebDriver** [[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)][[4](https://www.testdevlab.com/blog/top-20-software-testing-automation-frameworks-for-web-and-mobile-in-2025)]. В 2026 году он по-прежнему занимает около 26% рынка QA-инструментов, так как на нем написаны миллионы тестов в крупных компаниях [[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)]. 
    Однако писать на "голом" Selenium сложно, поэтому в Java-мире используют надстройку **Selenide**. Она убирает многословность кода и автоматически обрабатывает ожидания элементов. (Для C# активно внедряют Playwright [[4](https://www.testdevlab.com/blog/top-20-software-testing-automation-frameworks-for-web-and-mobile-in-2025)][[6](https://medium.com/sdet-insights/comparing-uft-developer-vs-playwright-vs-selenium-for-enterprise-applications-in-2025-eab71ba8208a)]).
*   **API-тестирование:**
    Золотой стандарт для Java — библиотека **REST Assured**. Это мощнейший инструмент, который позволяет отправлять запросы и проверять сложные JSON/XML ответы с помощью красивого, цепочечного синтаксиса (например, `given().when().get().then().statusCode(200)`).
*   **Тест-раннеры:**
    В Java используются классические движки **JUnit 5** или **TestNG**, а в C# — **NUnit** или **xUnit**. Они требуют чуть больше времени на настройку, но дают полный контроль над процессом запуска тысяч тестов [[6](https://medium.com/sdet-insights/comparing-uft-developer-vs-playwright-vs-selenium-for-enterprise-applications-in-2025-eab71ba8208a)].

---

### Вне конкуренции: Мобильная автоматизация и ИИ

Независимо от того, какой из трех языков вы выберете, есть инструменты, которые работают поверх всех экосистем:

1. **Мобильное тестирование (Appium):** Если вы решите тестировать iOS и Android, вы столкнетесь с Appium [[2](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)][[8](https://testlio.com/blog/test-automation-statistics/)]. Это стандарт де-факто (своего рода Selenium для смартфонов). Вы можете писать тесты для Appium на TS, Python или Java [[8](https://testlio.com/blog/test-automation-statistics/)][[9](https://sandra-parker.medium.com/mobile-app-automation-testing-4dd814138c2b)].
2. **AI-инструменты (Тренд 2026):** Индустрия активно внедряет ИИ [[3](https://www.practitest.com/assets/pdf/stot-2025.pdf)][[8](https://testlio.com/blog/test-automation-statistics/)]. Около 16% команд уже используют AI-агентов [[8](https://testlio.com/blog/test-automation-statistics/)]. Такие инструменты, как **GitHub Copilot** или встроенные AI-помощники в IDE, сегодня не заменяют AQA, но помогают за секунды генерировать Page Object классы на любом из трех языков.

---

### Резюме для самопроверки

Как Junior AQA, вы не должны пытаться выучить всё. Посмотрите вакансии в вашем регионе или в компаниях мечты и выберите свой путь:
1. Хотите тестировать **сложный UI и быть ближе к Frontend**? Ваш выбор: *TypeScript + Playwright/Cypress*.
2. Хотите быстро стартовать, тестировать **API, базы данных, бэкенд или тот же UI**? Ваш выбор: *Python + Pytest + Requests + Selenium/Playwright*.
3. Хотите стабильную работу в **банках и крупных корпорациях**? Ваш выбор: *Java + Selenide + REST Assured + JUnit*.

Именно в таких "пакетах" инструменты встречаются в реальных проектах. 

Если эта картина мира вам понятна, мы можем смело переходить к **Блоку 3: Инфраструктура, CI/CD и Аналитика**. Там мы узнаем, где и как вся эта красота запускается после того, как вы написали код!

Sources
1. [Automated Functional Testing: Why And How To Do It](https://luxequality.com/blog/automated-functional-testing/)
2. [Playwright vs Cypress vs Selenium in 2026: Which E2E Testing Framework to Choose](https://devtoolswatch.com/en/playwright-vs-cypress-vs-selenium-2026)
3. [STATE OF TESTING™ - REPORT 2025](https://www.practitest.com/assets/pdf/stot-2025.pdf)
4. [Top 20 Software Testing Automation Frameworks for Web and Mobile in 2025](https://www.testdevlab.com/blog/top-20-software-testing-automation-frameworks-for-web-and-mobile-in-2025)
5. [Top 10 QA Automation Tools for Startups in 2026](https://blog.greenroots.info/top-10-qa-automation-tools-for-startups-in-2026)
6. [Comparing UFT Developer vs Playwright vs Selenium for Enterprise Applications in 2025](https://medium.com/sdet-insights/comparing-uft-developer-vs-playwright-vs-selenium-for-enterprise-applications-in-2025-eab71ba8208a)
7. [How to Reduce Testing Time and Effort with Test Automation](https://www.testdevlab.com/blog/reduce-time-and-effort-with-automated-testing)
8. [Top 30+ Test Automation Statistics in 2025](https://testlio.com/blog/test-automation-statistics/)
9. [Mobile App Automation Testing](https://sandra-parker.medium.com/mobile-app-automation-testing-4dd814138c2b)
