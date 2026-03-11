# Блок 1: Погружение в новую роль
## Тема 3: Жизненный цикл автотеста (ATLC) на практике

В ручном тестировании вы наверняка привыкли к STLC (Software Testing Life Cycle). В мире автоматизации существует свой аналогичный процесс — **ATLC (Automation Testing Life Cycle)** [[1](https://www.browserstack.com/guide/automation-testing-life-cycle)][[2](https://medium.com/@alexraii/a-comprehensive-guide-to-the-automation-testing-life-cycle-d27602a388a0)]. В глобальном смысле он включает анализ бюджета, выбор инструментов, настройку серверов и написание фреймворка [[3](https://www.softwaretestingo.com/automation-testing-life-cycle-atlc/)]. 

Однако для Junior AQA-инженера, который приходит в уже настроенный проект, этот цикл сужается до очень понятной, но строгой ежедневной рутины. Она состоит из трех основных шагов: **анализ ручного кейса**, **написание/отладка скрипта** и **поддержка (Maintenance) написанного кода**. Давайте разберем каждый из них.

---

### 1. Анализ ручного тест-кейса: Готов ли он к автоматизации?

Перед тем как написать первую строчку кода, автоматизатор должен открыть ручной тест-кейс (в Jira, Zephyr, TestRail) и провести его аудит [[4](https://bugbug.io/blog/software-testing/test-automation-maintenance/)][[5](https://s2-labs.com/blog/automation-testing-life-cycle/)]. Робот не умеет додумывать, у него нет интуиции, поэтому далеко не каждый ручной сценарий можно превратить в автотест. 

**Критерии идеального тест-кейса для автоматизации [[6](https://huddle.eurostarsoftwaretesting.com/choosing-the-right-test-cases-for-automation-a-practical-guide/)][[7](https://www.qamadness.com/an-effective-approach-to-selecting-test-cases-for-automation/)]:**
1. **Однозначность и предсказуемость результата (Deterministic):** В тест-кейсе не должно быть шагов вроде *«Убедитесь, что страница загрузилась достаточно быстро»* или *«Проверьте, что кнопка выглядит красиво»*. Для скрипта нужны жесткие условия: *«Подождать появления элемента X не более 5 секунд, проверить, что текст элемента равен Y»* [[7](https://www.qamadness.com/an-effective-approach-to-selecting-test-cases-for-automation/)].
2. **Стабильность функционала (Stability):** Нельзя автоматизировать фичи, которые всё еще активно разрабатываются. Если дизайн или API-контракт меняется каждый день, ваш скрипт будет падать при каждом запуске [[6](https://huddle.eurostarsoftwaretesting.com/choosing-the-right-test-cases-for-automation-a-practical-guide/)].
3. **Высокая частота использования (Repetitiveness):** Сценарий должен регулярно запускаться (например, входить в ежедневный Smoke- или Регресс-набор). Если кейс проверяется раз в полгода, автоматизировать его экономически невыгодно [[6](https://huddle.eurostarsoftwaretesting.com/choosing-the-right-test-cases-for-automation-a-practical-guide/)][[8](https://jayateerthk.medium.com/how-i-select-right-test-scenarios-for-automation-4a5320d4acd6)].
4. **Независимость (Isolation):** Хороший тест-кейс должен быть самодостаточным. Он должен сам подготавливать для себя данные и удалять их после завершения (TearDown) [[9](https://devops.com/test-automation-strategy-for-growing-software-teams/)]. Автотест не должен падать только потому, что предыдущий тест завершился с ошибкой.

> **Практический совет Junior AQA:** Если шаги в ручном кейсе описаны расплывчато, ваша задача — пойти к ручному QA или системному аналитику и уточнить конкретные локаторы, статус-коды и ожидаемые тексты, прежде чем писать код.

---

### 2. Разработка и локальная отладка скрипта

После того как кейс отобран, начинается этап разработки (Test Script Development & Execution) [[1](https://www.browserstack.com/guide/automation-testing-life-cycle)]. В 2026 году этот процесс выходит далеко за рамки простого перевода шагов на язык программирования.

**Как строится работа над скриптом:**
1. **Написание кода (Scripting):** Вы переводите шаги (Given/When/Then) в код, используя готовый фреймворк проекта. Здесь важно соблюдать правила чистого кода (DRY — Don't Repeat Yourself) [[10](https://medium.com/@testwithblake/a-comprehensive-guide-to-efficient-test-automation-maintenance-dcb55defa4c0)]. Вы не хардкодите тестовые данные (не пишете логины и пароли прямо в тесте), а выносите их в специальные конфигурационные файлы.
2. **Расстановка проверок (Assertions):** Автотест без `Assert` (проверки) — это просто скрипт, кликающий по экрану. Именно Assert определяет, пройден тест (Passed) или провален (Failed). Проверки должны быть точными (например, ожидаем от сервера HTTP-статус `201 Created` и `user_id` в теле ответа) [[7](https://www.qamadness.com/an-effective-approach-to-selecting-test-cases-for-automation/)].
3. **Локальная отладка (Debugging):** Написанный тест **почти никогда не работает с первого раза**. Страница может загружаться дольше, чем ожидает скрипт, или всплывающее окно перекроет нужную кнопку. 
На этом этапе Junior AQA запускает тест на своем компьютере, смотрит в консоль на ошибки, добавляет *явные ожидания (Explicit Waits)* и добивается того, чтобы скрипт прошел успешно 3-5 раз подряд без единого сбоя. Только после этого код отправляется (Push) в общий репозиторий [[9](https://devops.com/test-automation-strategy-for-growing-software-teams/)].

---

### 3. Поддержка (Maintenance): Жизнь после запуска

Существует популярный миф: *«Написал автотест, запустил и забыл»*. На самом деле всё ровно наоборот. Согласно исследованиям за 2025–2026 годы, поддержка существующих тестов (Test Maintenance) — это самый дорогой, сложный и регулярный этап ATLC, на который инженеры тратят до 40% своего времени [[4](https://bugbug.io/blog/software-testing/test-automation-maintenance/)][[11](https://codoid.com/automation-testing/test-automation-maintenance-costs-smart-ways-to-reduce/)].

**Почему тесты «протухают» и падают (Maintenance Burden) [[4](https://bugbug.io/blog/software-testing/test-automation-maintenance/)][[11](https://codoid.com/automation-testing/test-automation-maintenance-costs-smart-ways-to-reduce/)]:**
* **Эволюция продукта (Evolving UI/API):** Фронтенд-разработчик переименовал класс у кнопки, или бэкендер добавил новое обязательное поле в JSON-запрос. Приложение работает корректно, но старый автотест об этом не знает — он ищет старый селектор и падает [[4](https://bugbug.io/blog/software-testing/test-automation-maintenance/)].
* **Проблемы с окружением и данными:** Кто-то из команды вручную зашел в базу данных и удалил профиль, под которым ваш автотест логинился каждый день [[4](https://bugbug.io/blog/software-testing/test-automation-maintenance/)].
* **Flaky tests (Плавающие тесты):** Главный кошмар автоматизатора. Это тесты, которые то проходят (Passed), то падают (Failed) без каких-либо изменений в коде. Причины: сетевые задержки, асинхронность современных SPA-приложений (страница еще не отрендерилась, а скрипт уже кликает), конфликты параллельного запуска [[4](https://bugbug.io/blog/software-testing/test-automation-maintenance/)][[11](https://codoid.com/automation-testing/test-automation-maintenance-costs-smart-ways-to-reduce/)].

**Как бороться с проблемами поддержки (Best Practices) [[4](https://bugbug.io/blog/software-testing/test-automation-maintenance/)][[10](https://medium.com/@testwithblake/a-comprehensive-guide-to-efficient-test-automation-maintenance-dcb55defa4c0)]:**
* **Правило «Разбитых окон»:** Если тест начал падать или стал flaky (плавающим), его нужно чинить немедленно или отключать (ставить метку `@Skip` или `@Ignore`). Если в отчетах CI/CD каждый день будет светиться красный цвет от сломанных тестов, команда разработчиков просто перестанет им доверять и будет игнорировать результаты [[10](https://medium.com/@testwithblake/a-comprehensive-guide-to-efficient-test-automation-maintenance-dcb55defa4c0)].
* **Динамические данные:** Вместо использования статических пользователей, скрипт должен сам регистрировать нового уникального пользователя через API перед началом проверки, а в конце — удалять его [[9](https://devops.com/test-automation-strategy-for-growing-software-teams/)].
* **Актуализация вместе с кодом:** Как только разработчики берут в работу новую фичу, изменяющую старый функционал, AQA должен запланировать время на рефакторинг (обновление) связанных автотестов [[9](https://devops.com/test-automation-strategy-for-growing-software-teams/)].

---

### Резюме для самопроверки

1. **Не всё подлежит автоматизации.** Для написания скрипта тест-кейс должен быть абсолютно предсказуемым, стабильным и часто повторяемым (например, регрессионным) [[6](https://huddle.eurostarsoftwaretesting.com/choosing-the-right-test-cases-for-automation-a-practical-guide/)][[7](https://www.qamadness.com/an-effective-approach-to-selecting-test-cases-for-automation/)].
2. **Отладка — это основа.** Тест нельзя коммитить в общий проект, пока вы не добьетесь его 100% стабильного прохождения на вашей локальной машине.
3. **Поддержка (Maintenance) неизбежна.** Автотесты ломаются из-за изменения UI, API, проблем с данными и сетью. Главная задача автоматизатора — быстро чинить "упавшие" тесты и безжалостно бороться с Flaky tests, чтобы сохранить доверие команды к результатам проверок [[4](https://bugbug.io/blog/software-testing/test-automation-maintenance/)][[10](https://medium.com/@testwithblake/a-comprehensive-guide-to-efficient-test-automation-maintenance-dcb55defa4c0)].

В следующем блоке мы перейдем к более технической части и узнаем, как правильно строить архитектуру автотестов, чтобы минимизировать боль на этапе поддержки!

Sources
1. [Automation Testing Life Cycle (ATLC): Importance and Stages](https://www.browserstack.com/guide/automation-testing-life-cycle)
2. [A Comprehensive Guide to the Automation Testing Life Cycle](https://medium.com/@alexraii/a-comprehensive-guide-to-the-automation-testing-life-cycle-d27602a388a0)
3. [Automation Testing Life Cycle ( ATLC )](https://www.softwaretestingo.com/automation-testing-life-cycle-atlc/)
4. [Test Automation Maintenance - A Comprehensive Guide for 2026](https://bugbug.io/blog/software-testing/test-automation-maintenance/)
5. [Automation Testing Life Cycle: A Complete Guide](https://s2-labs.com/blog/automation-testing-life-cycle/)
6. [Choosing the Right Test Cases for Automation: A Practical Guide](https://huddle.eurostarsoftwaretesting.com/choosing-the-right-test-cases-for-automation-a-practical-guide/)
7. [An Effective Approach to Selecting Test Cases for Automation](https://www.qamadness.com/an-effective-approach-to-selecting-test-cases-for-automation/)
8. [How I Select Right Test Scenarios For Automation ?](https://jayateerthk.medium.com/how-i-select-right-test-scenarios-for-automation-4a5320d4acd6)
9. [Test Automation Strategy for Growing Software Teams](https://devops.com/test-automation-strategy-for-growing-software-teams/)
10. [A Comprehensive Guide to Efficient Test Automation Maintenance](https://medium.com/@testwithblake/a-comprehensive-guide-to-efficient-test-automation-maintenance-dcb55defa4c0)
11. [Test Automation Maintenance Costs: Smart Ways to Reduce](https://codoid.com/automation-testing/test-automation-maintenance-costs-smart-ways-to-reduce/)
