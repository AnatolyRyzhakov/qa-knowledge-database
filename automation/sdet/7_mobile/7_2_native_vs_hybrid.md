# 📘 Глава 7. Mobile Automation Deep Dive
## Тема 7.2. Контексты: Native vs Hybrid vs Web (Context switching)

### Предисловие:
Понимание архитектуры мобильных приложений (Native, Web, Hybrid) и механизмов переключения между ними (Context Switching) — это фундаментальный навык, который проверяется на сертификациях уровня **ISTQB Mobile Application Testing (CT-MAT)** [1]. 

В 2026 году граница между "нативным" и "веб" контентом окончательно размылась. Большинство Enterprise-приложений (банки, e-commerce) являются гибридными: основной каркас написан на Swift/Kotlin, а динамические экраны (например, соглашения, акции, каталоги) подтягиваются "на лету" через встроенные браузеры (WebViews).

### Введение: Три кита мобильной архитектуры (Стандарты ISTQB)
Согласно учебному плану ISTQB CT-MAT, QA-инженер обязан различать архитектуру приложений для выбора правильной стратегии автоматизации [1]:
1.  **Native (Нативные):** Приложения написаны на языках платформы (Swift/Objective-C для iOS, Kotlin/Java для Android). Используют стандартные UI-компоненты ОС (кнопки, списки). Тестируются в контексте `NATIVE_APP`.
2.  **Mobile Web (Веб-приложения):** Обычные сайты, адаптированные под мобильный экран и запущенные в мобильном браузере (Safari/Chrome). Тестируются полностью через Selenium/Playwright или Appium в контексте `WEBVIEW`.
3.  **Hybrid (Гибридные):** Нативная "оболочка" (Native Shell), внутри которой встроен невизуальный браузер (WebView), рендерящий HTML/JS/CSS код. Тестирование требует постоянного переключения (Context Switching) между `NATIVE_APP` и `WEBVIEW` [2][4].

---

### Часть 1. Анатомия WebView: Как это работает "под капотом"

Когда Appium работает с гибридным приложением, он не может взаимодействовать с HTML-элементами внутри WebView напрямую через UIAutomator2 или XCUITest (они понимают только нативные XML-деревья мобильной ОС).

Для решения этой задачи Appium 2.0 использует архитектуру **Проксирования сессий (Session Proxying)** [2].
*   **На Android:** Когда вы переключаете контекст на WebView, Appium Server "под капотом" запускает встроенный **ChromeDriver** и перенаправляет все ваши W3C-команды (клики, поиск по CSS/XPath) ему [2][3].
*   **На iOS:** Appium использует `ios-webkit-debug-proxy` и проксирует команды к **SafariDriver**, который умеет общаться с движком WebKit внутри iOS-приложения.

**Критическое требование инфраструктуры:** 
Чтобы ChromeDriver или SafariDriver смогли подключиться к WebView, разработчики приложения обязаны включить режим отладки при сборке тестового `.apk` или `.ipa` [3].
*   *Android:* В коде приложения должно быть `WebView.setWebContentsDebuggingEnabled(true)`.
*   *iOS:* На самом устройстве должен быть включен переключатель `Web Inspector` (Settings $\rightarrow$ Safari $\rightarrow$ Advanced) [2].

---

### Часть 2. Переключение контекстов (Context Switching)

В Appium управление контекстами осуществляется через команды `driver.contexts` и `driver.switch_to.context()`.

```python
# Пример для Appium (Python)
def test_hybrid_checkout_flow(driver):
    # 1. Нативный шаг: кликаем на иконку корзины
    driver.find_element(AppiumBy.ACCESSIBILITY_ID, "cart_icon").click()
    
    # 2. Переход на экран оплаты (который является WebView)
    # Получаем список всех доступных контекстов [4]
    contexts = driver.contexts 
    print(contexts) # Выведет:['NATIVE_APP', 'WEBVIEW_com.mycompany.app']
    
    # 3. Переключаемся в WebView (Динамический поиск)
    webview_context = next(c for c in contexts if "WEBVIEW" in c)
    driver.switch_to.context(webview_context)
    
    # 4. Теперь мы работаем как в обычном браузере (используем CSS-селекторы!) [2]
    driver.find_element(AppiumBy.CSS_SELECTOR, "input#credit-card").send_keys("4111...")
    driver.find_element(AppiumBy.CSS_SELECTOR, "button.submit-pay").click()
    
    # 5. КРИТИЧЕСКИ ВАЖНО: Возврат в нативный контекст для продолжения теста [4]
    driver.switch_to.context("NATIVE_APP")
    assert driver.find_element(AppiumBy.ID, "success_native_banner").is_displayed()
```

---

### Часть 3. Специфика 2026 года: React Native и Flutter

В 2026 году классические "Гибридные" приложения (Cordova, Ionic) активно вытесняются фреймворками нового поколения, архитектура которых ломает старые подходы тестирования.

1.  **React Native:**
    Несмотря на то, что код пишется на JavaScript, React Native "транспилирует" его в **нативные компоненты** (Native Views) ОС. 
    *   *Подход:* Вам **не нужно** переключаться в `WEBVIEW`. Приложение тестируется целиком в `NATIVE_APP` контексте. Главная задача SDET — заставить разработчиков прописывать атрибуты `testID` в React-компонентах, которые превратятся в `accessibilityIdentifier` на устройствах.
2.  **Flutter (Dart):**
    Самый сложный кейс. Flutter не использует ни нативные UI-компоненты, ни WebView. Он рисует весь интерфейс самостоятельно (как игру) с помощью графического движка Skia/Impeller. Стандартный Appium увидит только один гигантский пустой квадрат на экране.
    *   *Подход:* Senior SDET должен установить специальный плагин и драйвер **Appium Flutter Driver**. В скрипте появится третий вид контекста — `FLUTTER`, который позволяет взаимодействовать с внутренним деревом виджетов Flutter.

---

### Часть 4. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns (Красные флаги на ревью)
1.  **Хардкод имени контекста:** Написание `driver.switch_to.context("WEBVIEW_1")`. Порядок и нумерация WebView зависят от ОС и версии устройства. Приложение может подгружать скрытые WebView (например, для рекламы), и индекс сместится. Всегда ищите контекст динамически по имени пакета `WEBVIEW_com.your.app` [2][4].
2.  **Чрезмерное переключение (Context Bouncing):** Переключение контекстов (`switch_to.context()`) — это тяжелая сетевая операция, которая заставляет Appium пересоздавать прокси-сессию с ChromeDriver [3]. Если вы переключаетесь туда-обратно 10 раз за тест, время прогона увеличится на 15-20 секунд. Сначала сделайте все нативные шаги, затем переключитесь в Web, сделайте все веб-шаги, и вернитесь обратно [3].
3.  **Попытка использовать XPath WebView в нативном контексте:** Если вы забыли переключить контекст обратно на `NATIVE_APP` и пытаетесь искать нативную кнопку по `accessibility_id`, тест упадет с `NoSuchElementException`, так как ChromeDriver ничего не знает о нативной ОС [2].

#### ✅ Best Practices (Стандарты 2026)
1.  **Умные ожидания (Wait for Context):** WebView не появляется мгновенно при переходе на новый экран. Если вы запросите `driver.contexts` сразу после клика, вы получите только `['NATIVE_APP']`. Senior SDET пишет кастомное ожидание (Custom Waiter), которое опрашивает `driver.contexts` до тех пор, пока массив не станет больше одного элемента (или пока не выйдет таймаут).
2.  **Синхронизация ChromeDriver (Matching Versions):** Самая частая проблема пайплайнов в гибридном тестировании — ошибка `SessionNotCreatedException: This version of ChromeDriver only supports Chrome version X`. В 2026 году SDET использует автоматические менеджеры (например, Appium capability `chromedriverExecutableDir` в связке с авто-загрузчиками), чтобы Appium сам скачивал ChromeDriver, версия которого строго совпадает с версией WebView (Chrome) на Android-эмуляторе [3].
3.  **Shadow DOM внутри WebView:** Если гибридное приложение использует Web Components внутри своего WebView, тестировщик должен применять те же правила, что и в Playwright (пробитие теневого дерева), используя CSS-инъекции или JS-скрипты, так как классический Appium XPath не пробивает Shadow-Root.

***

### 📚 Sources (Источники)
1. [ISTQB Mobile Application Testing Syllabus & Guidelines | ProcessExam](https://www.processexam.com/istqb/istqb-ct-mat-certification-exam-syllabus)
2. [Mastering Hybrid and Web App Testing with Appium: Insider Tips | Medium (Jun 2025)](https://medium.com/@ntiinsd/mastering-hybrid-and-web-app-testing-with-appium-insider-tips-and-tricks-for-2025-25b9488fc696)
3. [Appium: Mobile Automation Essentials & Hybrid Testing Best Practices | Dhiraj Das (Nov 2025)](https://www.dhirajdas.dev/blog/appium-mobile-automation-essentials)
4. [How to Test and Automate Hybrid Apps with Appium | TestGrid](https://docs.testgrid.io/automating-android-hybrid-apps/)
