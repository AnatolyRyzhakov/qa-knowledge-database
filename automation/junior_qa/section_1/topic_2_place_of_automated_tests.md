# Блок 1: Погружение в новую роль
## Тема 2: Место автотестов в разработке и Пирамида тестирования

В предыдущей теме мы выяснили, что автотесты — это инструмент для предотвращения старых ошибок (регрессии) и ускорения работы. Но когда именно они пишутся и запускаются? Чтобы автотесты приносили пользу, они должны быть правильно встроены в **SDLC (Software Development Life Cycle — жизненный цикл разработки ПО)**. 

Современная разработка живет короткими спринтами (Agile) и непрерывными релизами (DevOps). В этой теме мы разберем, как автоматизатору не стать «бутылочным горлышком» всего процесса и зачем индустрия придумала «Пирамиду тестирования».

---

### 1. Смена парадигмы: Концепция Shift-Left (Сдвиг влево)

Исторически тестирование находилось в самом конце цикла разработки: *Сбор требований → Дизайн → Написание кода → **Тестирование** → Релиз*. 
В этой классической (Waterfall) модели тестировщики ждали, пока разработчики полностью закончат фичу, и только потом начинали ее проверять и писать скрипты. В реалиях 2026 года с микросервисами и ежедневными релизами такой подход приводит к катастрофе — критические баги находятся за день до релиза [[1](https://www.virtuosoqa.com/post/shift-left-testing-early-with-the-sdlc)].

Индустрия перешла на концепцию **Shift-Left Testing (Сдвиг тестирования влево)** [[2](https://www.trigyn.com/insights/shift-left-and-shift-everywhere-embedding-quality-throughout-sdlc)]. 

Если представить этапы разработки в виде линии слева направо (где слева — планирование, а справа — релиз), то Shift-Left означает перенос обеспечения качества на самые ранние этапы (левую сторону) [[1](https://www.virtuosoqa.com/post/shift-left-testing-early-with-the-sdlc)].

**Как это работает для AQA-инженера на практике:**
1. **Автоматизация начинается до UI:** Вы не ждете, пока фронтендер нарисует красивые кнопки. Как только готов контракт (описание) API, вы уже пишете автотесты на логику работы бэкенда [[1](https://www.virtuosoqa.com/post/shift-left-testing-early-with-the-sdlc)].
2. **Анализ требований:** AQA-инженер участвует в планировании (Planning) и оценивает истории пользователей (User Stories) на «тестопригодность». Если логику нельзя автоматизировать — требования возвращаются на доработку [[2](https://www.trigyn.com/insights/shift-left-and-shift-everywhere-embedding-quality-throughout-sdlc)].
3. **Экономия бюджета:** Статистика показывает, что баг, найденный и исправленный на этапе требований или раннего написания кода, обходится компании в **100 раз дешевле**, чем тот же баг, найденный в уже готовом интерфейсе или на Production [[3](https://www.thedroidsonroids.com/blog/shift-left-testing-guide)].

---

### 2. Пирамида тестирования: Компас автоматизатора

Чтобы понимать, *какие именно* тесты нужно писать, в индустрии используется **Пирамида автоматизации тестирования (Test Automation Pyramid)** — концептуальная модель, созданная Майком Коном и ставшая золотым стандартом распределения тестов [[4](https://saucelabs.com/resources/blog/mobile-automated-testing-pyramid)][[5](https://www.parasoft.com/blog/testing-automation-pyramids-for-software-development/)].

Пирамида визуализирует идеальные пропорции автотестов в проекте. Классическая модель состоит из трех уровней:

#### Уровень 1 (База): Unit-тесты (Модульные)
Самый нижний, широкий уровень пирамиды [[6](https://testomat.io/blog/testing-pyramid-role-in-modern-software-testing-strategies/)].
* **Кто пишет:** Разработчики (иногда AQA с хорошим знанием архитектуры кода).
* **Что тестируют:** Изолированные кусочки кода (функции, методы, классы) без обращения к базе данных или сети [[7](https://capacity.so/blog/software-testing-best-practices)].
* **Особенности:** Они невероятно быстрые (выполняются за миллисекунды), дешевые в написании и их должно быть очень много (около 70% от всех тестов в проекте) [[7](https://capacity.so/blog/software-testing-best-practices)].

#### Уровень 2 (Середина): API / Интеграционные тесты
Золотая жила Junior/Middle AQA инженера.
* **Кто пишет:** AQA-инженеры и разработчики.
* **Что тестируют:** Работу приложения "под капотом" (без графического интерфейса). Как компоненты системы (микросервисы, базы данных) общаются между собой [[7](https://capacity.so/blog/software-testing-best-practices)][[8](https://medium.com/@merisstupar11/decoding-the-importance-of-api-automation-testing-27c1cf79a0e7)].
* **Особенности:** Занимают около 20% тестового покрытия [[7](https://capacity.so/blog/software-testing-best-practices)]. Работают быстро, высоконадежны (не падают из-за того, что дизайнер поменял цвет кнопки) и позволяют проверять бизнес-логику напрямую [[8](https://medium.com/@merisstupar11/decoding-the-importance-of-api-automation-testing-27c1cf79a0e7)].

#### Уровень 3 (Вершина): E2E / UI-тесты (End-to-End)
Самая узкая часть пирамиды [[7](https://capacity.so/blog/software-testing-best-practices)].
* **Кто пишет:** AQA-инженеры.
* **Что тестируют:** Полный путь пользователя (открыл браузер, кликнул, положил в корзину, оплатил) [[5](https://www.parasoft.com/blog/testing-automation-pyramids-for-software-development/)].
* **Особенности:** Должны составлять не более 10% всех тестов [[7](https://capacity.so/blog/software-testing-best-practices)]. Они очень медленные, ресурсоемкие и хрупкие (flaky). На этом уровне автоматизируют **только критические пути (Happy Paths)** и самые важные бизнес-флоу [[7](https://capacity.so/blog/software-testing-best-practices)].

> **Антипаттерн: "Рожок мороженого" (Ice-Cream Cone)**
> Это частая ошибка команд, где мануальные тестировщики только начинают автоматизировать. Они берут все свои ручные тест-кейсы и пишут на них тяжелые UI-тесты через браузер, игнорируя API и Unit-уровни. Пирамида переворачивается (широкая вершина из UI и полное отсутствие Unit базы). Итог: прогон тестов занимает 5 часов, они постоянно ломаются, и поддержка съедает всё время автоматизатора.

---

### 3. Направления автоматизации: С чем вы будете работать

Современный AQA не ограничивается только кликами по экрану. В зависимости от платформы, автоматизация делится на несколько инженерных направлений:

#### 1. Web UI Автоматизация
Классика, с которой часто начинают обучение. Скрипт управляет реальным (или скрытым — headless) браузером: ищет элементы на странице (по селекторам), кликает, вводит текст и считывает результаты. Требует умения работать с DOM-деревом и ожиданиями (waiters), чтобы тесты не падали из-за долгой загрузки страницы.

#### 2. API / Backend Автоматизация
Главный тренд последних лет [[8](https://medium.com/@merisstupar11/decoding-the-importance-of-api-automation-testing-27c1cf79a0e7)]. Тестирование отправкой HTTP-запросов (GET, POST, PUT, DELETE) напрямую к серверу и валидацией ответов (JSON/XML форматов), статусов ответов (200 OK, 404, 500) и структуры данных [[9](https://testfort.com/blog/api-testing-automation-how-and-why-automate-api-testing)]. 
*Почему это важно:* API-тесты легко встраиваются в CI/CD-пайплайны, не зависят от готовности фронтенда (согласно подходу Shift-Left) и работают в десятки раз быстрее UI [[8](https://medium.com/@merisstupar11/decoding-the-importance-of-api-automation-testing-27c1cf79a0e7)][[9](https://testfort.com/blog/api-testing-automation-how-and-why-automate-api-testing)].

#### 3. Mobile UI Автоматизация
Автоматизация взаимодействия с мобильными приложениями (Android/iOS). 
* Включает работу с нативными (Native), гибридными (Hybrid) и мобильными веб-приложениями [[4](https://saucelabs.com/resources/blog/mobile-automated-testing-pyramid)].
* Тесты запускаются не только на реальных физических устройствах (что дорого), но и на программных Эмуляторах (Android) и Симуляторах (iOS) [[4](https://saucelabs.com/resources/blog/mobile-automated-testing-pyramid)].

#### 4. Нефункциональная автоматизация (Обзорно для Junior QA)
Для расширения кругозора важно знать, что автотесты пишут не только для проверки бизнес-логики:
* **Performance Testing (Нагрузочное тестирование):** Скрипты, эмулирующие тысячи одновременных пользователей, чтобы проверить, "не упадет" ли сервер (инструменты типа JMeter, Gatling, k6) [[9](https://testfort.com/blog/api-testing-automation-how-and-why-automate-api-testing)][[10](https://goreplay.org/blog/api-testing-best-practices/)].
* **Security Testing (Безопасность):** Автоматические сканеры (SAST/DAST), которые пытаются "взломать" API, подставляя вредоносный код (SQL-инъекции) или проверяя уязвимости авторизации [[10](https://goreplay.org/blog/api-testing-best-practices/)][[11](https://www.testingxperts.com/blog/api-security-testing/)].

---

### Резюме для самопроверки

1. **Концепция Shift-Left** учит нас не ждать окончания разработки, а внедрять автоматизацию и контроль качества как можно раньше (анализ требований, написание тестов по контрактам API).
2. **Пирамида тестирования** показывает, что тесты должны быть сбалансированы. Чем ниже тест к коду (Unit, API), тем он быстрее, надежнее и дешевле.
3. Огромное количество UI-тестов при полном отсутствии других уровней — это антипаттерн **"Рожок мороженого"**, который убьет проект долгой и дорогой поддержкой.
4. В арсенале современного AQA-инженера **API-тесты занимают центральное место**, так как они стабильны, независимы от графического интерфейса и идеально подходят для быстрых (DevOps) релизов.

В следующей теме мы разберем весь путь конкретного автотеста: от чтения ручного тест-кейса до анализа его падения. Мы изучим **Жизненный цикл автотеста (ATLC)** на практике глазами Junior AQA!

Sources
1. [Shift-Left Testing - Types, Benefits, and Best Practices](https://www.virtuosoqa.com/post/shift-left-testing-early-with-the-sdlc)
2. [Shift Left and Shift Everywhere: Embedding Quality Throughout the SDLC](https://www.trigyn.com/insights/shift-left-and-shift-everywhere-embedding-quality-throughout-sdlc)
3. [What Is Shift Left Testing Strategy: Complete Guide](https://www.thedroidsonroids.com/blog/shift-left-testing-guide)
4. [Best Practices for Effective Mobile Testing: The Modern Mobile Automated Testing Pyramid](https://saucelabs.com/resources/blog/mobile-automated-testing-pyramid)
5. [How Is the Test Automation Pyramid Used in Software Development?](https://www.parasoft.com/blog/testing-automation-pyramids-for-software-development/)
6. [Test Pyramid Explained: A Strategy for Modern Software Testing for Agile Teams](https://testomat.io/blog/testing-pyramid-role-in-modern-software-testing-strategies/)
7. [Top Software Testing Best Practices to Ensure Quality](https://capacity.so/blog/software-testing-best-practices)
8. [Decoding the Importance of API Automation Testing](https://medium.com/@merisstupar11/decoding-the-importance-of-api-automation-testing-27c1cf79a0e7)
9. [API Testing Automation: How And Why Automate API Testing](https://testfort.com/blog/api-testing-automation-how-and-why-automate-api-testing)
10. [Understanding API Testing Requirements and Documentation](https://goreplay.org/blog/api-testing-best-practices/)
11. [API Security Testing: A Step-by-Step Guide](https://www.testingxperts.com/blog/api-security-testing/)
