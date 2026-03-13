# Блок 3: Инфраструктура и Аналитика (Жизнь тестов на сервере)
## Тема 8: Анализ результатов и "плавающие" тесты (Flaky tests)

Вы приходите утром на работу, открываете отчет CI/CD конвейера и видите, что из 100 автотестов упало 5. Что делать дальше? Бежать к разработчикам и заводить 5 баг-репортов? Нет. Сначала вы должны провести **анализ (Triage)** и выяснить причину падения. 

Главная проблема автоматизации заключается в том, что автотест может упасть не только из-за ошибки в продукте, но и из-за того, что он **Flaky (плавающий)**. Давайте разберем этот феномен.

---

### 1. Что такое Flaky tests и почему они разрушают проекты?

**Плавающий тест (Flaky test)** — это автотест, который при запуске на одной и той же версии кода и в одном и том же окружении выдает разные результаты: сейчас он упал (Failed), вы перезапустили его ничего не меняя — и он прошел успешно (Passed) [[3](https://testrigor.com/blog/flaky-tests/)][[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)].

**Почему это катастрофа для команды:**
1.  **Потеря доверия (Эффект "Мальчика и волков"):** Если тесты регулярно падают просто так, разработчики привыкают к "красному" цвету CI/CD пайплайна [[1](https://www.accelq.com/blog/flaky-tests/)]. Они начинают думать: *"А, это опять автоматизаторы написали кривой тест, мой код нормальный, отправляем в релиз"*. В итоге настоящий баг уходит к пользователям [[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)].
2.  **Замедление релизов:** Команда вынуждена по 3-4 раза перезапускать весь конвейер, тратя часы времени серверов в надежде, что в этот раз тесты "позеленеют" [[1](https://www.accelq.com/blog/flaky-tests/)][[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)].

---

### 2. Четыре главные причины "плавающих" тестов

Чтобы вылечить болезнь, нужно знать ее возбудителя. В 2026 году выделяют четыре основные причины flakiness:

*   **Причина 1: Асинхронность и проблемы ожидания (Timing / Race conditions)**
    Современные интерфейсы (React, Vue) загружаются частями. Автотест работает в 100 раз быстрее человека. Если вы написали код без правильных ожиданий, скрипт попытается кликнуть по кнопке "Купить" в ту же миллисекунду, когда страница открылась [[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)]. Кнопка визуально уже есть, но логика к ней еще не привязалась. *Итог: тест падает* [[5](https://testdino.com/blog/playwright-flaky-tests/)].
*   **Причина 2: Зависимость от тестовых данных (Shared State)**
    Тест №1 создает пользователя `Ivan` и проверяет его удаление. Но если тесты запускаются параллельно, Тест №2 может попытаться авторизоваться под этим же `Ivan` в тот момент, когда первый тест его уже удалил [[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)]. *Итог: второй тест падает случайным образом*.
*   **Причина 3: Нестабильное окружение и сеть (Environment instability)**
    Во время прогона теста сервер с базой данных "моргнул", или сторонний API (например, сервис оплаты Stripe) отвечал на 2 секунды дольше обычного [[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)]. *Итог: тест упал по таймауту*.
*   **Причина 4: Хрупкие локаторы**
    Вы привязали поиск кнопки к селектору `.mt-4.flex.button-blue`. Завтра дизайнер поменял цвет на красный, класс стал `button-red`. *Итог: логика работает, но тест не нашел кнопку* [[6](https://www.ranorex.com/blog/flaky-tests/)].

---

### 3. Как проводить анализ (Triage) упавших тестов

В 2026 году никто не ищет причину падения, вслепую вчитываясь в полотна текста в консоли. Для этого есть профессиональные инструменты.

#### Инструменты отчетности (Allure Report, ReportPortal, Testmo)
Это красивые интерактивные дашборды, которые собирают данные о прогонах [[7](https://ai-4.dev/software-testing/top-automated-test-reporting-tools/)]. 
*   **История прогонов:** Главное оружие Junior AQA. Если вы видите, что тест упал, посмотрите его историю. Если он падает только по вторникам или через раз — это Flaky [[2](https://agiletest.app/automated-test-report/)][[5](https://testdino.com/blog/playwright-flaky-tests/)]. Если он был зеленым 3 месяца, а после вчерашнего коммита разработчика стал 100% красным — **это настоящий баг**.

#### Анализ Артефактов (Trace Viewer и Видео)
Когда тесты бегают на удаленном сервере без монитора (в Headless-режиме), фреймворки сохраняют "улики" падения [[5](https://testdino.com/blog/playwright-flaky-tests/)]:
1.  **Скриншоты и Видеозаписи:** Прикрепляются к Allure-отчету автоматически при падении [[8](https://www.youtube.com/watch?v=n7ZziUey-jk)]. Вы открываете видео и видите своими глазами, что на странице вылез огромный баннер "Технические работы", который перекрыл нужную кнопку.
2.  **Trace Viewer (Машина времени для тестов):** Индустриальный стандарт от фреймворка Playwright [[8](https://www.youtube.com/watch?v=n7ZziUey-jk)][[9](https://playwright.dev/docs/trace-viewer-intro)]. Это специальный файл, который записывает всё, что происходило в браузере. Вы открываете Trace Viewer и можете ползунком перемещаться по каждому шагу теста вперед-назад [[9](https://playwright.dev/docs/trace-viewer-intro)]. Вы видите точный снимок DOM-дерева, какие сетевые запросы уходили и какие ошибки падали в консоль браузера на *каждой миллисекунде* теста [[9](https://playwright.dev/docs/trace-viewer-intro)]. 

---

### 4. Стратегии борьбы: Как управлять Flaky-тестами

Победить flakiness на 100% невозможно, но индустрия выработала строгие правила, как держать этот показатель на уровне менее 5% [[3](https://testrigor.com/blog/flaky-tests/)][[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)].

1.  **Правило Карантина (Stop the Line):**
    Если тест признан плавающим, его нужно **немедленно отключить** (пометить тегом `@Skip` или `@Ignore`) и убрать из основного CI/CD конвейера в так называемый "карантин" [[6](https://www.ranorex.com/blog/flaky-tests/)]. Он не должен блокировать работу разработчиков. Чинить его вы будете локально на своем компьютере.
2.  **Умные ожидания (Explicit Waits) вместо жестких пауз:**
    Никогда не пишите в коде `sleep(5000)` (подождать 5 секунд). Если сервер ответит за 1 секунду, вы потеряете 4 секунды впустую. Если за 6 секунд — тест упадет. Современные фреймворки имеют встроенные умные ожидания: *«Ждать, пока кнопка не станет кликабельной, но не дольше 10 секунд»* (`toBeVisible()`, `waitForResponse()`) [[5](https://testdino.com/blog/playwright-flaky-tests/)].
3.  **Изоляция данных:**
    Каждый тест должен сам генерировать для себя уникального пользователя через API перед началом (Setup) и удалять его после завершения (Teardown) [[6](https://www.ranorex.com/blog/flaky-tests/)]. Никакого хардкода и общих баз данных [[5](https://testdino.com/blog/playwright-flaky-tests/)]!
4.  **Механизм Retries (Авто-перезапуск):**
    В CI/CD настраивают автоматический перезапуск упавшего теста (обычно 1-2 раза) [[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)][[9](https://playwright.dev/docs/trace-viewer-intro)]. Если во время теста моргнула сеть, на второй попытке он пройдет. **Важно:** если тест прошел со 2-й попытки, CI/CD загорится зеленым, но в отчете этот тест будет помечен желтым флажком ("Flaky") [[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)][[9](https://playwright.dev/docs/trace-viewer-intro)]. Ваша задача — всё равно забрать его в починку, так как retry скрывает симптом, но не лечит болезнь [[3](https://testrigor.com/blog/flaky-tests/)][[5](https://testdino.com/blog/playwright-flaky-tests/)].

---

### Резюме для самопроверки

1.  **Flaky tests** — это тесты, которые то проходят, то падают без изменений в коде. Они опасны тем, что убивают доверие команды к результатам автоматизации [[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)].
2.  Основные причины нестабильности: попытки кликнуть по элементу до его загрузки (проблема ожиданий), конфликты тестовых данных и нестабильность сети [[4](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)][[5](https://testdino.com/blog/playwright-flaky-tests/)].
3.  Для расследования причин падения используются **Отчеты (Allure)** для просмотра истории и **Артефакты (Trace Viewer, Видео)** для покадрового анализа того, что происходило в браузере в момент сбоя [[8](https://www.youtube.com/watch?v=n7ZziUey-jk)][[9](https://playwright.dev/docs/trace-viewer-intro)].
4.  Плавающие тесты нужно немедленно отправлять в **карантин** (отключать в CI/CD), чтобы они не блокировали релизы [[6](https://www.ranorex.com/blog/flaky-tests/)].
5.  Использование автоматических перезапусков (Retries) спасает конвейер от падения из-за сетевых сбоев, но злоупотреблять ими нельзя — нестабильный код нужно переписывать с использованием динамических ожиданий [[5](https://testdino.com/blog/playwright-flaky-tests/)].

Поздравляю! Мы закрыли самый сложный, аналитический блок работы автоматизатора. Впереди у нас финальная, 9-я тема — **Использование ИИ (AI) в работе Junior AQA**. Посмотрим, как нейросети в 2026 году помогают писать код и анализировать тесты?

Sources
1. [Flaky Tests in 2026: How to Identify, Fix, and Prevent Them](https://www.accelq.com/blog/flaky-tests/)
2. [Automated Test Report in 2026: Manual to Automated Testing](https://agiletest.app/automated-test-report/)
3. [Flaky Tests – How to Get Rid of Them](https://testrigor.com/blog/flaky-tests/)
4. [How to Handle Flaky Tests: What Your Test System Should Do](https://www.qawolf.com/blog/what-your-system-should-do-with-a-flaky-test)
5. [Playwright flaky tests: detection, causes, and fixes](https://testdino.com/blog/playwright-flaky-tests/)
6. [Flaky Tests in Automation: Strategies for Reliable Automated Testing](https://www.ranorex.com/blog/flaky-tests/)
7. [Top 10 Automated Test Reporting Tools in 2025](https://ai-4.dev/software-testing/top-automated-test-reporting-tools/)
8. [Allure Report with Playwright 🔥 Screenshots, Videos, Trace Viewer & Retries | Complete Tutorial](https://www.youtube.com/watch?v=n7ZziUey-jk)
9. [Trace viewer](https://playwright.dev/docs/trace-viewer-intro)
