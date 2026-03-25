# 📘 Глава 6. Playwright и Advanced Web
## Тема 6.4. Специфика Web: Shadow DOM, прямая работа с Local/Session Storage, SEO Testing (SSR)

### Предисловие:
Современный фронтенд 2026 года — это не просто набор HTML-тегов. Приложения строятся на изолированных Web-компонентах (Shadow DOM), хранят критическое состояние в памяти браузера и используют гибридный рендеринг (SSR) для поисковых роботов. 

Для Senior SDET понимание этих механизмов обязательно. Инструмент Playwright предоставляет нативные API для работы с этими слоями, заменяя десятки "костылей", которые приходилось писать в эпоху Selenium.

### Введение: Почему изменился подход к UI-тестированию?
Согласно стандартам **ISO/IEC 25010:2023**, качество веб-продукта оценивается не только по функциональности, но и по **Compatibility (Совместимости)** и **Discoverability (Находимости/SEO)**. Тестировщик больше не может просто кликать по кнопкам. Он должен уметь пробивать инкапсуляцию компонентов (Shadow DOM), управлять токенами в хранилище браузера (Storage) и проверять то, как страницу видят поисковые роботы (Googlebot).

---

### Часть 1. Shadow DOM (Теневой DOM) и инкапсуляция

**Shadow DOM** — это стандарт браузеров для создания Web-компонентов (Web Components). Он позволяет разработчикам скрывать HTML и CSS внутри компонента (например, кастомного `<video-player>`), чтобы стили основного сайта их не сломали [1].

**Проблема старых фреймворков (Selenium):**
Обычные CSS-селекторы и XPath не могут заглянуть внутрь Shadow DOM. Если кнопка спрятана в "тени", классический `driver.find_element()` выдаст `NoSuchElementException`. Инженерам приходилось писать сложные JavaScript-инъекции (через `execute_script`), чтобы "проткнуть" теневой корень (`shadowRoot`)[2].

**Решение Playwright 2026 (Automatic Piercing):**
Playwright поддерживает работу с Shadow DOM "из коробки" [1][2]. Его встроенный движок локаторов автоматически "пробивает" (pierces) открытые теневые деревья.

```typescript
// Если кнопка <button class="submit"> лежит глубоко внутри Shadow DOM
// Playwright найдет ее автоматически, без дополнительных усилий:
await page.locator('.submit').click();

// А лучше использовать семантические локаторы (Best Practice 2026)[3]
await page.getByRole('button', { name: 'Submit' }).click();
```

---

### Часть 2. Манипуляция состоянием: Local Storage и Session Storage

Как мы обсуждали в Теме 5.3, прохождение UI-авторизации перед каждым тестом — это трата времени. Современные приложения хранят JWT-токены и сессии не только в Cookie, но и в **LocalStorage / SessionStorage / IndexedDB** [4][5].

В Playwright есть два архитектурных способа прямой работы с хранилищем.

#### Способ 1: Прямая инъекция токенов (Bypassing UI)
Если вы получили токен через API (Тема 5.1), вы можете напрямую положить его в `localStorage` страницы *до* того, как она будет отрисована, используя `page.add_init_script()` или `page.evaluate()` [4].

```python
async def test_dashboard_with_injected_token(page, api_token):
    # Внедряем JS-код, который положит токен в LocalStorage браузера
    await page.add_init_script(f"""
        window.localStorage.setItem('auth_token', '{api_token}');
    """)
    
    # При переходе на страницу, React/Vue прочитает токен и сразу пустит нас в систему
    await page.goto("/dashboard")
    await expect(page.locator(".welcome-message")).to_be_visible()
```

#### Способ 2: Клонирование сессии через `storageState`
Это золотой стандарт 2026 года для тестирования MFA (Multi-Factor Authentication) [5][6]. Вы логинитесь *один раз* в специальном `global-setup` скрипте, а затем Playwright делает слепок (Snapshot) всех Cookies и LocalStorage в JSON-файл.
```python
# Сохранение слепка (в setup-файле)
await page.context.storage_state(path="auth_state.json")

# Мгновенное восстановление слепка для 1000 тестов[5]
context = await browser.new_context(storage_state="auth_state.json")
```

---

### Часть 3. SEO Testing и Server-Side Rendering (SSR)

В 2026 году SDET отвечает не только за баги, но и за видимость продукта для бизнеса [7]. Приложения на *Next.js*, *Nuxt* или *SvelteKit* используют **SSR (Серверный рендеринг)**, чтобы поисковики могли прочитать контент без выполнения JavaScript [7][8].

#### 1. Валидация SSR (Проверка того, что видит робот)
Поисковые роботы (crawlers) часто работают без JS или с ограниченным движком. Чтобы проверить, корректно ли отработал SSR, Senior SDET отключает JavaScript в браузере Playwright.

```python
async def test_ssr_hydration_and_content(browser):
    # Создаем контекст с ОТКЛЮЧЕННЫМ JavaScript [8]
    context = await browser.new_context(java_script_enabled=False)
    page = await context.new_page()
    
    await page.goto("https://my-ssr-app.com/blog/article-1")
    
    # Если контент рендерится только на клиенте (CSR), тест упадет.
    # Если SSR настроен верно, текст будет в чистом HTML.
    await expect(page.locator("article .content")).to_contain_text("Важный SEO текст")
```

#### 2. Проверка Meta-тегов и Open Graph
Потеря тегов `description` или `og:image` при релизе приводит к падению трафика. Автотесты Playwright идеально подходят для валидации скрытого `<head>` документа [7][9].

```python
async def test_seo_meta_tags(page):
    await page.goto("/product/123")
    
    # Валидация Canonical URL
    canonical = page.locator('link[rel="canonical"]')
    await expect(canonical).to_have_attribute("href", "https://site.com/product/123")
    
    # Валидация Open Graph (для соцсетей) [9]
    og_title = page.locator('meta[property="og:title"]')
    await expect(og_title).to_have_attribute("content", "Купить Смартфон X")
    
    # Проверка robots.txt (через API-запрос внутри Playwright) [7]
    response = await page.request.get("/robots.txt")
    assert "Disallow: /admin/" in await response.text()
```

---

### Часть 4. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns (Красные флаги на ревью)
1.  **Использование XPath для Shadow DOM:** XPath физически не поддерживает прохождение сквозь `shadow-root` границы [2]. Попытки написать сложный XPath для Web Components всегда заканчиваются провалом.
2.  **Зависимость тестов от UI-авторизации:** Выполнение шагов "ввести логин, пароль, нажать войти" в `beforeEach` каждого теста. Это тратит часы времени в CI/CD и увеличивает процент Flaky-падений [4][5].
3.  **Игнорирование SEO в E2E:** Оставлять проверку мета-тегов на ручных тестировщиков или сторонние сервисы (Lighthouse), которые запускаются уже *после* релиза. В 2026 году SEO-тесты должны быть встроены в PR-пайплайн (Shift-Left SEO) [7].

#### ✅ Best Practices (Стандарты 2026)
1.  **Role-Based Selectors:** При работе с Shadow DOM всегда используйте `getByRole`, `getByLabel` или `getByTestId`. Playwright сам найдет элемент, независимо от того, насколько глубоко в теневом дереве он спрятан [2][3].
2.  **Storage State Factory:** Если в системе есть 3 роли (Admin, Manager, User), создайте скрипт, который генерирует 3 файла: `admin_state.json`, `manager_state.json`, `user_state.json`. В тестах просто инжектируйте нужный файл в нужный `BrowserContext` [5][6].
3.  **Проверка Hydration:** При тестировании SSR-приложений проверяйте не только статический HTML, но и момент "гидратации" (когда JS оживляет страницу). Убедитесь, что после загрузки JS кнопка "Купить" не мерцает и не исчезает (предотвращение Cumulative Layout Shift) [8].

***

### 📚 Sources (Источники)
1. [Shadow DOM in Playwright: Quick Guide | nonstopio (Sep 2025)](https://blog.nonstopio.com/shadow-dom-in-playwright-quick-guide-5d13fc13323b?gi=7b47aad3d524)
2. [Handling Shadow DOM and Web Components with Playwright | Testdock.io (Mar 2025)](https://www.testdock.io/articles/handling-shadow-dom-and-web-components-with-playwright)
3. [Shadow DOM Testing That Doesn't Flake (Using Playwright) | Medium (Aug 2025)](https://medium.com/@erik.amaral/shadow-dom-testing-that-doesnt-flake-using-playwright-1c9313d086d3)
4. [Using Playwright's storageState for Persistent Authentication | Medium (Feb 2025)](https://medium.com/@byteAndStream/using-playwrights-storagestate-for-persistent-authentication-f5b7384995d6)
5. [Using Playwright's storageState | BrowserStack (Dec 2025)](https://www.browserstack.com/guide/playwright-storage-state)
6. [Automating MFA Testing with Playwright Storage State | Sogeti Labs (Dec 2025)](https://labs.sogeti.com/conquering-mfa-how-playwrights-built-in-storage-state-revolutionizes-multi-factor-authentication-testing/)
7.[End-to-End SEO Testing with Playwright and Lighthouse | Planet Argon (Feb 2025)](https://blog.planetargon.com/blog/entries/end-to-end-seo-testing-with-playwright-and-lighthouse)
8.[Client-Side Rendering (CSR) vs. Server-Side Rendering (SSR) in Automation Testing | Medium (Aug 2024)](https://medium.com/@merisstupar11/client-side-rendering-csr-vs-server-side-rendering-ssr-in-automation-testing-e5390b4e3f13)
9.[How to Test meta tags using Playwright | Sergio Xalambrí](https://sergiodxa.com/tutorials/test-meta-tags-using-playwright)
