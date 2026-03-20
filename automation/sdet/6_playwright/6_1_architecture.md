# 📘 Глава 6. Playwright и Advanced Web
## Тема 6.1. Архитектура Playwright: Browser Contexts, изоляция сессий

### Предисловие:
В 2026 году этот инструмент от Microsoft стал абсолютным стандартом индустрии для автоматизации сложных веб-приложений (SPA, PWA, Micro-frontends), практически вытеснив Selenium из новых Enterprise-проектов [5][6]. 

Чтобы понимать, почему Playwright работает в разы быстрее и стабильнее своих предшественников, Senior SDET должен детально разбираться в его внутренней архитектуре, особенно в концепции **Browser Contexts (Контекстов браузера)**.

### Введение: Контекст стандартов и Архитектурный сдвиг
Согласно международному стандарту **ISO/IEC 25010** (в части *Performance Efficiency* и *Reliability*), тестовые фреймворки должны эффективно расходовать ресурсы (CPU/RAM) и гарантировать предсказуемость результатов [2]. 

Исторически (в эпоху Selenium WebDriver) для изоляции тестов приходилось запускать отдельный экземпляр (процесс) браузера для *каждого* теста. Если в CI/CD пайплайне нужно было запустить 100 тестов параллельно, серверу приходилось поднимать 100 процессов Chrome, что приводило к катастрофическому исчерпанию оперативной памяти (OOM) и падению серверов [1][5]. 

Playwright решает эту проблему на уровне архитектуры с помощью протокола **WebSocket** и **Контекстов**.

---

### Часть 1. Фундамент: WebSockets vs HTTP

Главное архитектурное отличие Playwright от Selenium заключается в протоколе связи [6].
*   **Selenium (HTTP):** Использует RESTful HTTP-запросы. Каждое действие (найти элемент, кликнуть) — это отдельный HTTP-запрос от вашего кода к драйверу, а от драйвера к браузеру. Это создает огромные задержки (Latency)[5][6].
*   **Playwright (WebSocket + CDP):** Playwright устанавливает **одно постоянное двунаправленное WebSocket-соединение** напрямую с браузерным движком (используя Chrome DevTools Protocol - CDP). Команды передаются мгновенно в виде JSON-сообщений. Это позволяет Playwright не только отправлять команды, но и "слушать" браузер в реальном времени (перехватывать сетевые запросы, события консоли) [4][6].

---

### Часть 2. Иерархия Playwright: Browser $\rightarrow$ Context $\rightarrow$ Page

Архитектура Playwright строится на трех уровнях [3][4]:

1.  **Browser (Браузер):** Физический процесс (Chrome, Firefox, WebKit). Запускается *один раз* и требует много памяти.
2.  **Browser Context (Контекст):** Легковесная, полностью изолированная сессия внутри одного процесса Browser [3][4]. Это аналог окна "Инкогнито". Создание нового контекста занимает миллисекунды и почти не потребляет CPU/RAM[4].
3.  **Page (Страница):** Вкладка внутри контекста.

**Почему это гениально для SDET?**
Вам больше не нужно открывать и закрывать сам браузер. Playwright запускает один экземпляр Chrome в фоне, а затем для каждого из 1000 тестов мгновенно "спавнит" (создает) новый `BrowserContext`. Каждый контекст имеет **собственные, изолированные**:
*   Cookies (Куки)
*   Local Storage / Session Storage
*   IndexedDB и Кэш
*   Состояние авторизации (Auth State) [3][5].

---

### Часть 3. Изоляция сессий и решение проблемы Flaky Tests

Главная причина плавающих (Flaky) тестов в UI-автоматизации — **утечка состояния (State Leakage)**. Если Тест А добавил товар в корзину, а Тест Б начал выполняться в том же браузере и увидел непустую корзину, Тест Б упадет [3].

Благодаря автоматической генерации `BrowserContext` для каждого теста, Playwright гарантирует **100% изоляцию**. Тесты стартуют в абсолютно стерильной среде, даже если они выполняются параллельно на одном ядре процессора [2][3].

#### Продвинутый паттерн: Multi-Role Testing (Тестирование нескольких ролей)
Поскольку контексты дешевые и изолированные, вы можете создавать их вручную прямо внутри одного теста. Это идеальный паттерн для тестирования чатов, систем совместного редактирования (Google Docs) или ролевых моделей (Admin vs User) [5].

```python
# Пример Playwright (Python)
import asyncio
from playwright.async_api import async_playwright

async def test_admin_can_block_user():
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        
        # Создаем ДВА абсолютно независимых контекста в одном браузере
        admin_context = await browser.new_context()
        user_context = await browser.new_context()
        
        admin_page = await admin_context.new_page()
        user_page = await user_context.new_page()
        
        # Логинимся под разными ролями. Их куки НЕ пересекаются! [3][5]
        await admin_page.goto("https://app.com/login?role=admin")
        await user_page.goto("https://app.com/login?role=user")
        
        # Админ блокирует юзера
        await admin_page.click("#block-user-btn")
        
        # Проверяем, что у юзера в ту же секунду появилось уведомление (WebSockets тест)
        assert await user_page.locator("#banned-banner").is_visible()
```

---

### Часть 4. Переиспользование состояния (StorageState Pattern)

Абсолютная изоляция — это хорошо, но если каждый из 1000 тестов будет проходить долгий процесс UI-авторизации (ввод логина, пароля, 2FA-кода), прогон займет часы [1]. 

В 2026 году стандартом является использование механизма **`storageState`** [3].

**Как это работает:**
1.  Специальный Setup-скрипт запускается один раз перед всеми тестами. Он авторизуется через UI или API и сохраняет слепок контекста (Cookies и LocalStorage) в JSON-файл[3].
2.  Все остальные автотесты при создании своего `BrowserContext` просто "подкладывают" этот JSON-файл.
3.  *Итог:* Тесты стартуют мгновенно уже будучи авторизованными, но при этом остаются изолированными друг от друга [3][4].

```python
# 1. Сохранение состояния (Setup)
context = await browser.new_context()
page = await context.new_page()
await page.goto("/login")
# ... шаги авторизации ...
await context.storage_state(path="auth_state.json") # Сохраняем слепок

# 2. Использование в тестах (Мгновенный логин)
# Playwright инжектирует куки в новый изолированный контекст
context = await browser.new_context(storage_state="auth_state.json")
```

---

### Часть 5. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns (Красные флаги на ревью)
1.  **Создание нового Browser для каждого теста:** Написание `await p.chromium.launch()` внутри каждого теста или фикстуры `scope="function"`. Это убьет процессор CI/CD сервера и замедлит тесты в 10 раз [1][4].
2.  **Запуск тестов в одной вкладке (Page):** Использование одной страницы для выполнения цепочки тестов (где Тест 2 зависит от того, что накликал Тест 1). Это нарушает принцип независимости тестов (Test Independence) [2].
3.  **Очистка кук вручную:** Написание шагов вида `await page.context.clear_cookies()` в блоке Teardown. Это "костыли" эпохи Selenium. В Playwright контекст просто уничтожается целиком (`await context.close()`) по окончании теста, забирая с собой все данные [3].

#### ✅ Best Practices (Стандарты 2026)
1.  **Фикстуры уровня Context:** В связке с Pytest, `browser` должен быть фикстурой уровня `session`, а `context` и `page` — фикстурами уровня `function`. (Официальный плагин `pytest-playwright` делает это под капотом по умолчанию) [3].
2.  **Эмуляция мобильных устройств через Контексты:** `BrowserContext` позволяет задавать параметры эмуляции (User-Agent, Viewport, Geolocation, Timezone) [5]. Вы можете запустить один тест как iPhone 15 Pro, а другой параллельно — как Desktop Chrome, используя один физический браузер.
3.  **Изоляция сетевых правил:** В Playwright перехват сети (Network Interception) настраивается на уровне контекста. Если вы замокаете (Mock) API-ответ со статусом 500 для проверки обработки ошибок, это повлияет *только* на текущий тест, не сломав параллельно идущие соседние проверки [5].

***

### 📚 Sources (Источники)
1. [Playwright Scalability: How to Run Thousands of Tests in Parallel at Enterprise Speed (Codehall, Nov 2025)](https://www.codehall.in/playwright-scalability-how-to-run-thousands-of-tests-in-parallel-at-enterprise-speed/)
2. [15 Best Practices for Playwright testing in 2026 (BrowserStack, Dec 2025)](https://www.browserstack.com/guide/playwright-best-practices)
3. [Playwright BrowserContext: What It Is, Why It Matters, and How to Configure It (DEV Community, Feb 2026)](https://dev.to/johnnyv5g/playwright-browsercontext-what-it-is-why-it-matters-and-how-to-configure-it-3gi8)
4. [Playwright Architecture: Complete Visual Guide to How it Works (TestDino, Mar 2026)](https://testdino.com/blog/playwright-architecture/)
5. [Playwright vs Selenium 2025: Comparing Test Automation and Scraping (Browserless, Sep 2025)](https://www.browserless.io/blog/playwright-vs-selenium-2025-browser-automation-comparison)
6. [Playwright vs. Selenium: A 2026 Architecture Review (DEV Community, Jan 2026)](https://dev.to/deepak_mishra_35863517037/playwright-vs-selenium-a-2026-architecture-review-347d)
