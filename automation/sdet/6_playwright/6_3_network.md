# 📘 Глава 6. Playwright и Advanced Web
## Тема 6.3. Манипуляции сетью: Network Interception, мокирование API-запросов

### Предисловие:
В 2026 году Senior SDET не просто проверяет "счастливый путь" (Happy Path), но и тестирует устойчивость фронтенда к сбоям бэкенда. Это достигается за счет нативного управления сетевым слоем (Network Interception)

### Введение: Контекст стандартов (ISO 25010 и ISTQB)
Согласно стандарту **ISO/IEC 25010**, качество системы включает атрибут **Reliability** (Надежность), а именно **Fault Tolerance** (Отказоустойчивость) — способность системы сохранять работоспособность при сбоях в компонентах. Если бэкенд возвращает `500 Internal Server Error`, фронтенд не должен "падать" с белым экраном, он должен показать красивую заглушку пользователю.

В руководстве **ISTQB Advanced Level Test Automation Engineer (TAE)** подчеркивается необходимость изоляции тестируемого компонента с помощью **Заглушек и Моков (Stubs & Mocks)** для обеспечения стабильности среды [2][5]. Playwright позволяет делать это прямо "из коробки" на уровне браузерного движка (через Chrome DevTools Protocol).

---

### Часть 1. Архитектура: Как работает Interception в Playwright

Playwright позволяет перехватить *любой* сетевой запрос (XHR, Fetch, загрузку картинок, скриптов), исходящий из страницы, **до того**, как он покинет браузер [3]. 

Основной метод — `page.route(url_pattern, handler)` [3]. Когда запрос совпадает с паттерном URL, Playwright ставит его на паузу и передает объект `Route` в вашу функцию-обработчик (Callback).

У вас есть три базовых пути работы с перехваченным запросом:
1.  **`route.continue()`**: Пропустить запрос к реальному серверу (как обычно) [1].
2.  **`route.abort()`**: Заблокировать запрос (сбросить соединение) [3].
3.  **`route.fulfill()`**: Замокать (подменить) ответ, вообще не обращаясь к серверу [4].

**Пример ускорения тестов (Блокировка аналитики и тяжелой статики):**
```python
async def test_page_load_fast(page):
    # Блокируем загрузку картинок и сторонних скриптов аналитики (Google Analytics)
    await page.route("**/*.{png,jpg,jpeg}", lambda route: route.abort())
    await page.route("**/google-analytics.com/**", lambda route: route.abort())
    
    await page.goto("https://my-app.com")
```

---

### Часть 2. Мокирование API (API Mocking)

Зависимость UI-тестов от нестабильных API или сложных тестовых данных (TDM) — главная причина flaky-тестов [2][4]. Мокирование позволяет фронтенду думать, что он общается с реальным сервером, получая при этом идеальные, детерминированные данные [1].

**Идеальный Use-Case:** Тестирование UI-компонента корзины с 10 000 товарами. Создавать 10 000 товаров в реальной БД через API — долго.
```python
import json

async def test_shopping_cart_rendering(page):
    # 1. Готовим синтетические данные
    mock_data =[{"id": 1, "name": "Fake Product", "price": 99.99}] * 10000
    
    # 2. Перехватываем запрос и подменяем ответ [4]
    async def handle_route(route):
        await route.fulfill(
            status=200,
            content_type="application/json",
            body=json.dumps(mock_data)
        )
        
    await page.route("**/api/v1/products", handle_route)
    
    # 3. Фронтенд получит наш mock_data за 1 миллисекунду
    await page.goto("/cart")
    await expect(page.locator(".product-item")).to_have_count(10000)
```

---

### Часть 3. Fault Injection: Симуляция сбоев и Edge Cases

Это любимый вопрос на интервью для Senior SDET: *"Как вы проверите, что при таймауте оплаты крутится лоадер, а при 500-й ошибке выводится красное уведомление?"*. Вызывать реальную 500-ю ошибку на сервере сложно. С Playwright мы симулируем её на лету (Edge Case Testing) [1][2].

**Симуляция 500 Internal Server Error:**
```python
async def test_payment_server_error(page):
    # При запросе на оплату возвращаем 500 ошибку
    await page.route("**/api/payments", lambda route: route.fulfill(status=500))
    
    await page.click("#buy-button")
    # Проверяем (Fault Tolerance), что UI обработал сбой корректно
    await expect(page.locator("#error-toast")).to_have_text("Server error, try again.")
```

**Симуляция медленного интернета (Timeouts):**
В 2026 году нельзя использовать жесткий `time.sleep()`. Для проверки лоадеров мы искусственно задерживаем API-ответ [1].
```python
import asyncio

async def handle_slow_route(route):
    await asyncio.sleep(5) # Задерживаем ответ на 5 секунд
    await route.continue() # Затем пропускаем запрос к реальному API

await page.route("**/api/heavy-data", handle_slow_route)
```

---

### Часть 4. Продвинутая техника: Modify Response (Патчинг ответов)

Иногда нам нужен реальный ответ от сервера, но мы хотим изменить (patch) в нем только одно поле [4]. Например, мы делаем реальный логин, но хотим "на лету" подменить роль пользователя с `USER` на `SUPER_ADMIN`, чтобы открыть скрытые элементы интерфейса [4].

```python
async def patch_user_role(route):
    # 1. Выполняем РЕАЛЬНЫЙ запрос к серверу
    response = await route.fetch()
    
    # 2. Получаем реальный JSON
    data = await response.json()
    
    # 3. Модифицируем (патчим) данные
    data["role"] = "SUPER_ADMIN"
    
    # 4. Возвращаем фронтенду модифицированный ответ [4]
    await route.fulfill(
        response=response,
        json=data
    )

await page.route("**/api/profile", patch_user_role)
```

---

### Часть 5. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns (Красные флаги на ревью)
1.  **Over-mocking (Чрезмерное мокирование):** Если 100% ваших тестов используют `route.fulfill()`, вы тестируете фронтенд в вакууме. Такие тесты не найдут баг, если разработчики бэкенда случайно переименуют поле в контракте API (например, `userName` на `user_name`). Отчет будет зеленым, а на Production всё сломается [1][2].
2.  **Загрязнение тестов:** Хардкод огромных JSON-строк прямо внутри тестовых методов `test_*.py`. Это нарушает читаемость кода [1].
3.  **Тестирование 3rd-party зависимостей наживую:** Попытка в тестах делать реальные транзакции через Stripe, PayPal или отправлять реальные SMS через Twilio. Вы исчерпаете лимиты и получите блокировку аккаунта [5].

#### ✅ Best Practices (Стандарты 2026)
1.  **Строгое мокирование 3rd-Party:** Всегда используйте Network Interception для изоляции сторонних сервисов (Third-party dependencies), которые вы не контролируете [5]. Тест должен проверять *только* то, что ваше приложение корректно отправляет данные в этот сервис.
2.  **Mocking for Edge Cases:** Используйте моки преимущественно для негативных сценариев (404, 500, таймауты), которые трудно или невозможно воспроизвести в реальном стейджинговом окружении [1]. Для Happy Path предпочтительнее интеграционные E2E прогоны с Data Seeding (Тема 5.1).
3.  **Использование HAR-файлов (HTTP Archive):** В Playwright 2026 года стандартом является запись реального сетевого трафика в `.har` файл один раз, а затем использование команды `page.route_from_har()` [3][4]. Это позволяет тестировать сложные интеграционные сценарии на "замороженном" слепке реальной сети (Replaying from HAR), не писать моки руками и легко обновлять их при смене API [4].

***

### 📚 Sources (Источники)
1. [Network Interception and Mocking in Playwright | by Manish Saini (Oct 2024)](https://medium.com/the-testing-hub/network-interception-and-mocking-in-playwright-3f490e91a2cb)
2. [11 Pivotal Best Practices for Playwright - Autify (Mar 2025)](https://autify.com/blog/playwright-best-practices/)
3. [Network - Playwright Official Documentation](https://playwright.dev/docs/network)
4. [Mock APIs - Playwright Official Documentation](https://playwright.dev/docs/mock)
5. [Best Practices - Playwright Official Documentation (Avoid testing third-party dependencies)](https://playwright.dev/docs/best-practices)
