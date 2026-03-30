# 📘 Глава 7. Mobile Automation Deep Dive
## Тема 7.3. Взаимодействие: W3C Actions API для сложных жестов

### Предисловие:
Взаимодействие с экраном устройства (тапы, свайпы, зум) исторически было "головной болью" SDET, так как платформы iOS и Android обрабатывали жесты по-разному. 

С выходом Appium 2.0 и полным переходом на стандарт **W3C WebDriver** правила игры кардинально изменились. В 2026 году старые классы признаны устаревшими, а индустрия перешла на унифицированный W3C Actions API

### Введение: Контекст стандартов и смерть `TouchAction`
В сертификации **ISTQB CT-MAT** подчеркивается, что мобильное приложение должно быть протестировано на предмет корректной обработки многопальцевых жестов (Multi-touch) и сложной навигации (Operability / Usability по стандарту ISO/IEC 25010)[2][3].

Долгое время автоматизаторы использовали классы `TouchAction` и `MultiTouchAction`. **В 2026 году эти классы официально удалены (Deprecated) из клиентских библиотек Appium (начиная с перехода на Selenium 4)** [5][7]. 
*Почему?* Они не были стандартизированы W3C, вели себя непредсказуемо на разных ОС и не поддерживали современные виды устройств (например, стилусы) [5]. 

На смену им пришел **W3C Actions API** — универсальный стандарт, основанный на строгой математике и координатах [2][5].

---

### Часть 1. Анатомия W3C Actions API: Как это работает?

W3C Actions API — это низкоуровневый, абстрактный интерфейс. Он оперирует не "свайпами" и "тапами", а **Источниками ввода (Input Sources)** и **Событиями (Events)** [3].

Чтобы построить любой жест, Senior SDET должен собрать последовательность (Sequence) из четырех базовых атомарных команд:
1.  **`pointerMove`** — перемещение пальца (указателя) в заданную координату [2].
2.  **`pointerDown`** — касание экрана (нажатие) [2].
3.  **`pause`** — удержание пальца на месте (критично для имитации человеческого взаимодействия) [2].
4.  **`pointerUp`** — отпускание экрана [2].

---

### Часть 2. Реализация базовых и сложных жестов (Python 2026)

В Python для работы с W3C Actions используются классы `ActionChains`, `ActionBuilder` и `PointerInput`. Мы явно указываем тип взаимодействия: `POINTER_TOUCH` [6][7].

#### 1. Классический Свайп (Swipe)
Свайп — это перемещение пальца от точки А к точке Б за определенное время.
```python
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.common.actions.action_builder import ActionBuilder
from selenium.webdriver.common.actions.pointer_input import PointerInput
from selenium.webdriver.common.actions import interaction

def perform_swipe(driver, start_x, start_y, end_x, end_y):
    # Создаем виртуальный "палец"
    touch_input = PointerInput(interaction.POINTER_TOUCH, "finger1")
    actions = ActionChains(driver)
    
    # Переопределяем дефолтную мышь на тач-указатель [7]
    actions.w3c_actions = ActionBuilder(driver, mouse=touch_input)
    
    # Строим последовательность (Sequence)
    actions.w3c_actions.pointer_action.move_to_location(start_x, start_y) # Навели палец
    actions.w3c_actions.pointer_action.click_and_hold()                   # Коснулись экрана
    actions.w3c_actions.pointer_action.move_to_location(end_x, end_y)     # Провели по экрану
    actions.w3c_actions.pointer_action.release()                          # Отпустили
    
    actions.perform() # Отправляем JSON-команду на Appium Server
```

#### 2. Drag and Drop (Перетаскивание)
Drag & Drop концептуально похож на свайп, но требует паузы (удержания) на стартовом элементе, чтобы приложение поняло, что мы хотим "схватить" объект, а не проскроллить страницу[1][4].

```python
def drag_and_drop(driver, element_to_drag, target_zone):
    # Получаем координаты центров элементов
    start = element_to_drag.location
    end = target_zone.location
    
    touch_input = PointerInput(interaction.POINTER_TOUCH, "finger1")
    actions = ActionChains(driver)
    actions.w3c_actions = ActionBuilder(driver, mouse=touch_input)
    
    actions.w3c_actions.pointer_action.move_to_location(start['x'], start['y'])
    actions.w3c_actions.pointer_action.click_and_hold()
    # Добавляем паузу 500ms, чтобы активировать Drag-режим приложения! [2]
    actions.w3c_actions.pointer_action.pause(0.5) 
    actions.w3c_actions.pointer_action.move_to_location(end['x'], end['y'])
    actions.w3c_actions.pointer_action.release()
    
    actions.perform()
```

#### 3. Multi-Touch: Pinch & Zoom (Масштабирование)
Это задача уровня Architect. Чтобы сделать Zoom, нам нужно создать **два независимых "пальца"** (Pointer Inputs) и заставить их двигаться *одновременно* в противоположных направлениях [1][3]. В W3C это достигается объединением двух `Sequence` в один `perform()`.

---

### Часть 3. Альтернатива W3C: Mobile Gestures Commands (Экстеншены)

Поскольку писать низкоуровневый код W3C для каждого свайпа трудоемко, драйверы XCUITest и UIAutomator2 поддерживают специальные скриптовые экстеншены (Mobile Commands). Они выполняются через `driver.execute_script()` [1][4].

**Преимущество:** Это высокоуровневые команды, которые работают "из коробки" и часто выполняются быстрее, так как транслируются напрямую в нативные API iOS/Android [1].

```python
# Элегантный Swipe Up (Прокрутка вниз) с помощью Mobile Commands [1][4]
driver.execute_script("mobile: swipeGesture", {
    "left": 100, "top": 100, "width": 200, "height": 200,
    "direction": "up",
    "percent": 0.75 # Проскроллить 75% от высоты указанного блока
})

# Drag and drop по ID элемента [4]
driver.execute_script("mobile: dragGesture", {
    "elementId": element.id,
    "endX": 500,
    "endY": 800
})
```
*Замечание Архитектора:* Mobile Commands платформозависимы. Набор аргументов для `mobile: swipe` в iOS (XCUITest) и Android (UIAutomator2) может отличаться. W3C Actions — полностью кроссплатформенный подход [1].

---

### Часть 4. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns (Красные флаги на ревью)
1.  **Использование `TouchAction` / `MultiTouchAction` в новом коде:** Использование устаревших (deprecated) классов. При попытке запустить такой код на Appium 2 + Selenium 4, тест упадет с ошибкой `AttributeError` или `NotImplementedError` [5][7].
2.  **Абсолютные координаты (Hardcoding):** Написание `move_to_location(150, 800)`. Этот тест пройдет на эмуляторе iPhone 15, но упадет на iPad или Android-планшете из-за разного разрешения экранов.
3.  **Игнорирование пауз (`pause`):** Отсутствие задержки между `pointerDown` и `pointerMove` превращает жест "Свайп" в "Флик" (Flick - резкий бросок). ОС может не распознать это как скролл [2].

#### ✅ Best Practices (Стандарты 2026)
1.  **Относительные координаты:** Senior SDET всегда вычисляет координаты динамически на основе размера экрана: `screen_size = driver.get_window_size()`. Идеальный старт свайпа — это `screen_size['width'] / 2` и `screen_size['height'] * 0.8` [6].
2.  **Обертка в Helper-классы:** Скройте сложный низкоуровневый код `ActionChains` внутри класса-помощника `GestureHelper`. Ваши UI-тесты (Test Layer) должны вызывать понятные методы: `GestureHelper.swipe_down_until_element_visible(element_locator)`.
3.  **Использование `scrollGesture` вместо `swipeGesture` для навигации:** Свайп (Swipe) — это слепое движение по координатам. Скролл (Scroll) — это контролируемое движение до элемента. Если вам нужно прокрутить список, чтобы найти кнопку, используйте `mobile: scrollGesture` (на Android) с передачей целевого локатора, чтобы драйвер сам остановил прокрутку в нужный момент[4].

***

### 📚 Sources (Источники)
1. [How to Automate Mobile Gestures With Appium | TestMu AI (Mar 2026)](https://www.testmuai.com/learning-hub/appium-gestures/)
2. [How to Automate Gesture Testing with Appium | Applitools (Updated 2026)](https://applitools.com/blog/how-to-automate-gesture-testing-appium/)
3. [Automating Complex Gestures with the W3C Actions API | HeadSpin (2025/2026 Architecture)](https://www.headspin.io/blog/automating-complex-gestures-with-the-w3c-actions-api)
4. [Gestures in Appium: Drag and Drop (W3C vs Mobile Commands) | Monfared.io (Mar 2024 / Updated 2026)](https://blog.monfared.io/posts/gestures-in-appium-part7-drag-and-drop/)
5. [The transition from Touch to W3C Actions in Selenium | The Green Report (Mar 2023 / 2026 relevance)](https://www.thegreenreport.blog/articles/the-transition-from-touch-to-w3c-actions-in-selenium/the-transition-from-touch-to-w3c-actions-in-selenium.html)
6. [Appium-Python-client TouchAction replacement with W3C PointerInput | Stack Overflow (Mar 2023)](https://stackoverflow.com/questions/75881566/appium-python-client-touchaction-class-is-being-deprecated-how-do-i-perform-pre)
7. [Appium Python Client Documentation (Deprecation of MultiAction/TouchAction) | PyPI](https://pypi.org/project/Appium-Python-Client/2.7.0/)
