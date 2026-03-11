# Блок 1: Погружение в новую роль
## Тема 1: Введение в автотестирование: от ручного к автоматизированному

Добро пожаловать в инженерию качества (Quality Engineering)! Переход от ручного тестирования (Manual QA) к автоматизированному (AQA) — это не просто изучение языка программирования. В первую очередь, это **смена образа мышления**. Ручной тестировщик мыслит сценариями использования и пользовательским опытом, а автоматизатор — архитектурой, надежностью проверок и оптимизацией рутины.

В 2026 году автоматизация тестирования стала стандартом индустрии: согласно исследованиям, внедрение автотестов в компаниях выросло на 85%, так как бизнес стремится к быстрым и частым релизам [[1](https://avoautomation.com/blog/test-cases-to-not-automate)]. Однако автоматизация не является волшебной таблеткой. Давайте разберем, как всё устроено на самом деле.

---

### 1. В чем реальная разница подходов (Плюсы и минусы)

Долгое время в IT-сообществе существовало противопоставление «Manual vs Automation». Сегодня, в 2026 году, доказано, что ни один из подходов не является универсальным [[2](https://www.getpanto.ai/blog/manual-testing-vs-automated-testing)]. Лучшие команды используют **гибридную стратегию**, где каждый инструмент решает свою задачу [[3](https://muuktest.com/blog/manual-testing-vs-automation-testing)], [[4](https://www.ranger.net/post/manual-vs-automated-testing-which-wins)].

#### Ручное тестирование (Manual QA)
Это процесс проверки ПО с участием человеческой интуиции, эмпатии и способности к адаптации [[5](https://www.virtuosoqa.com/post/automated-vs-manual-testing)]. 

**Плюсы:**
* **Эмпатия и UX:** Только человек может сказать, удобен ли интерфейс, приятна ли анимация и логичен ли флоу [[3](https://muuktest.com/blog/manual-testing-vs-automation-testing)].
* **Гибкость на ранних этапах:** Если продукт на стадии MVP (минимально жизнеспособный продукт) и интерфейс меняется каждый день, ручному тестировщику легко подстроиться [[6](https://www.testaces.com/manual-vs-automation-testing-guide.php)].
* **Исследовательское тестирование (Exploratory Testing):** Поиск нестандартных краевых случаев (edge cases), которые невозможно заранее прописать в скрипте [[6](https://www.testaces.com/manual-vs-automation-testing-guide.php)], [[7](https://bugbug.io/blog/test-automation/software-testing-best-practices/)].

**Минусы:**
* **Низкая скорость и плохая масштабируемость:** Невозможно вручную проверять сотни функций в нескольких браузерах перед каждым ежедневным релизом [[6](https://www.testaces.com/manual-vs-automation-testing-guide.php)], [[4](https://www.ranger.net/post/manual-vs-automated-testing-which-wins)].
* **Человеческий фактор:** При монотонном выполнении одних и тех же регрессионных тестов глаз «замыливается», и тестировщик может пропустить очевидный баг [[6](https://www.testaces.com/manual-vs-automation-testing-guide.php)].

#### Автоматизированное тестирование (AQA)
Это написание кода (скриптов), который управляет браузером, отправляет API-запросы или обращается к базе данных для автоматической проверки ожидаемого результата [[2](https://www.getpanto.ai/blog/manual-testing-vs-automated-testing)].

**Плюсы:**
* **Невероятная скорость и повторяемость:** Автотесты могут выполнить тысячи проверок за пару минут и делать это круглосуточно в фоновом режиме в CI/CD пайплайнах [[5](https://www.virtuosoqa.com/post/automated-vs-manual-testing)], [[4](https://www.ranger.net/post/manual-vs-automated-testing-which-wins)].
* **Быстрая обратная связь для разработчиков:** Разработчик узнает о том, что сломал старый функционал, через 5 минут после сохранения кода, а не через неделю [[5](https://www.virtuosoqa.com/post/automated-vs-manual-testing)], [[8](https://www.deviqa.com/blog/the-cost-of-ignoring-best-practices-in-test-automation/)].
* **Освобождение времени:** Забирая рутину (регресс), автотесты высвобождают время QA-инженеров для сложного исследовательского тестирования [[9](https://sqa.stackexchange.com/questions/381/what-is-meant-by-automated-tests-dont-find-new-bugs/721)].

**Минусы:**
* **Высокий порог входа и стоимость на старте:** Написание фреймворка, инфраструктуры и первых тестов требует времени и инженерных навыков (инструменты, серверы, CI/CD) [[2](https://www.getpanto.ai/blog/manual-testing-vs-automated-testing)], [[5](https://www.virtuosoqa.com/post/automated-vs-manual-testing)].
* **Дорогая поддержка (Maintenance):** Автотесты хрупкие. Изменился дизайн кнопки или структура базы данных — тест упадет (станет Flaky), и его придется чинить [[5](https://www.virtuosoqa.com/post/automated-vs-manual-testing)], [[10](https://testrigor.com/blog/10-quality-myths-busted/)].
* **Слепота к незапрограммированному:** Скрипт проверит только то, что вы ему написали. Если на половину экрана вылезет огромный баннер с ошибкой, но кнопка, которую ищет скрипт, кликабельна — тест пройдет успешно (если не настроены визуальные проверки) [[5](https://www.virtuosoqa.com/post/automated-vs-manual-testing)], [[9](https://sqa.stackexchange.com/questions/381/what-is-meant-by-automated-tests-dont-find-new-bugs/721)].

---

### 2. Что нужно автоматизировать, а что — бессмысленно?

Один из главных навыков Junior AQA — умение сказать **«Нет, мы не будем это автоматизировать»**. Попытка покрыть скриптами 100% тест-кейсов — это путь к катастрофе, так как поддержка такого объема тестов съест весь бюджет проекта [[1](https://avoautomation.com/blog/test-cases-to-not-automate)], [[10](https://testrigor.com/blog/10-quality-myths-busted/)].

#### ✅ ЧТО НУЖНО автоматизировать (Кандидаты с высоким ROI)
> [!NOTE]
> ROI (Return on Investment) - это коэффициент окупаемости вложений, который показывает финансовую эффективность перехода с ручного тестирования на автоматизированное

1. **Регрессионное тестирование (Regression):** Тесты функционала, который был разработан давно и работает стабильно. Это самая частая причина внедрения автоматизации — чтобы не проверять старое вручную перед каждым релизом [[1](https://avoautomation.com/blog/test-cases-to-not-automate)], [[11](https://testomat.io/blog/test-management-best-practices-and-tools/)].
2. **Smoke-тесты (Дымовое тестирование):** Критические пути пользователя (Critical Path) — логин, добавление в корзину, оплата. Если они не работают, релиз отменяется [[11](https://testomat.io/blog/test-management-best-practices-and-tools/)].
3. **Подготовка тестовых данных (Data Prep):** Скрипты, которые генерируют в базе данных 100 уникальных пользователей с разными ролями, чтобы ручные тестировщики могли сразу приступить к работе [[1](https://avoautomation.com/blog/test-cases-to-not-automate)], [[11](https://testomat.io/blog/test-management-best-practices-and-tools/)].
4. **Data-Driven тестирование:** Сценарии, где одно и то же действие нужно проверить с огромным количеством разных входных данных (например, валидация поля email с 50 разными форматами) [[11](https://testomat.io/blog/test-management-best-practices-and-tools/)].
5. **Нагрузочное тестирование:** Проверка того, как система ведет себя при 10 000 одновременных пользователей (человеческими ресурсами это смоделировать невозможно) [[11](https://testomat.io/blog/test-management-best-practices-and-tools/)].

#### ❌ ЧТО БЕССМЫСЛЕННО автоматизировать (Оставляем ручным QA)
1. **Новые фичи с нестабильным UI:** Если интерфейс или логика функции еще меняются в процессе спринта, написанный сегодня автотест завтра придется переписывать [[3](https://muuktest.com/blog/manual-testing-vs-automation-testing)], [[6](https://www.testaces.com/manual-vs-automation-testing-guide.php)].
2. **UX и UI-вкус (Subjective Validation):** Автоматика не скажет, контрастен ли текст на фоне, удобно ли расположена кнопка, и не раздражает ли пользователя анимация загрузки [[1](https://avoautomation.com/blog/test-cases-to-not-automate)], [[6](https://www.testaces.com/manual-vs-automation-testing-guide.php)].
3. **Одноразовые или редко используемые функции:** Если фича используется раз в год (например, генерация сложного годового отчета для админа), написать и поддерживать для неё автотест выйдет дороже, чем проверить вручную [[3](https://muuktest.com/blog/manual-testing-vs-automation-testing)], [[11](https://testomat.io/blog/test-management-best-practices-and-tools/)].
4. **Защитные системы (Капчи, 2FA, биометрия):** Смысл капчи — не пустить бота. Пытаться автоматизировать её прохождение — это борьба с ветряными мельницами (для тестов капчу просто отключают в тестовом окружении).

---

### 3. Популярные мифы об автоматизации: Разрушаем иллюзии

Вокруг автотестирования витает множество мифов. Как будущему инженеру, вам важно оперировать фактами, а не иллюзиями.

#### Миф 1: «Автотесты найдут много новых багов в новых фичах»
**Факт:** Это одно из самых больших заблуждений! Автотесты практически **не ищут новые баги**, они предотвращают появление старых (регрессию) [[12](https://blog.qatestlab.com/2024/04/11/test-automation-myths-where-the-truth-ends-and-the-myth-begins/)], [[9](https://sqa.stackexchange.com/questions/381/what-is-meant-by-automated-tests-dont-find-new-bugs/721)].
*Почему так происходит?* В индустрии есть важное правило: скрипты правильнее называть **«автоматизированными проверками» (Automated Checks)**, а не тестированием [[9](https://sqa.stackexchange.com/questions/381/what-is-meant-by-automated-tests-dont-find-new-bugs/721)]. Скрипт всегда идет по заранее известному (захардкоженному) пути. Если разработчик допускает логическую ошибку в новой фиче, скорее всего, AQA напишет тест, опираясь на те же ошибочные требования. 
Настоящие *новые* баги находит ручной тестировщик в процессе исследовательского тестирования (когда он делает то, что не предусмотрено сценарием) [[12](https://blog.qatestlab.com/2024/04/11/test-automation-myths-where-the-truth-ends-and-the-myth-begins/)], [[9](https://sqa.stackexchange.com/questions/381/what-is-meant-by-automated-tests-dont-find-new-bugs/721)]. Автотесты становятся барьером (сеткой), который не дает старым багам пролезть в Production [[12](https://blog.qatestlab.com/2024/04/11/test-automation-myths-where-the-truth-ends-and-the-myth-begins/)], [[13](https://imalittletester.com/2017/03/14/a-false-myth-automated-tests-dont-uncover-bugs/)].

#### Миф 2: «Автоматизация заменит ручных тестировщиков»
**Факт:** Автоматизация призвана освободить QA от рутины, а не уволить их. В 2026 году, даже с развитием ИИ (GenAI), ручные QA-инженеры крайне востребованы для проверки сложных бизнес-логик, оценки юзабилити и эмпатического взаимодействия с продуктом [[7](https://bugbug.io/blog/test-automation/software-testing-best-practices/)]. Без ручного QA-анализа AQA-инженеру просто нечего будет автоматизировать [[3](https://muuktest.com/blog/manual-testing-vs-automation-testing)].

#### Миф 3: «Автотесты — это 'написал и забыл' (Set-and-Forget)»
**Факт:** Код тестов — это такой же программный код продукта. Он подвержен техническому долгу, устареванию и ошибкам [[3](https://muuktest.com/blog/manual-testing-vs-automation-testing)], [[10](https://testrigor.com/blog/10-quality-myths-busted/)]. В реальности AQA-инженер тратит до 30-50% своего рабочего времени не на написание новых тестов, а на **поддержку (Maintenance) старых** [[5](https://www.virtuosoqa.com/post/automated-vs-manual-testing)]. Изменился ID элемента на странице? Тест упал — идем чинить. Сервер стал отвечать на секунду дольше? Тест отвалился по таймауту — идем чинить [[3](https://muuktest.com/blog/manual-testing-vs-automation-testing)]. 

#### Миф 4: «Чем больше автотестов, тем лучше качество (100% покрытие)»
**Факт:** Закон убывающей доходности отлично работает в тестировании. Достичь 100% покрытия физически и финансово невозможно [[12](https://blog.qatestlab.com/2024/04/11/test-automation-myths-where-the-truth-ends-and-the-myth-begins/)], [[10](https://testrigor.com/blog/10-quality-myths-busted/)]. Создание 5 000 UI-тестов приведет к тому, что их прогон будет занимать 5 часов, они будут постоянно случайным образом падать (flaky tests), и команда перестанет обращать внимание на их результаты [[10](https://testrigor.com/blog/10-quality-myths-busted/)]. В 2026 году фокус ставится на *умное покрытие* (Smart Coverage): тестируются только самые критичные бизнес-цепочки (Risk-based testing) [[1](https://avoautomation.com/blog/test-cases-to-not-automate)], [[10](https://testrigor.com/blog/10-quality-myths-busted/)].

---

### Резюме для самопроверки

Перед тем как двигаться к следующей теме, убедитесь, что вы усвоили ключевые тезисы:
1. Вы понимаете, что автоматизация — это не замена Manual QA, а отдельный инженерный инструмент для решения проблемы масштабируемости и скорости.
2. Вы знаете, что писать автотесты на нестабильный UI или пытаться проверять удобство дизайна скриптом — это плохая практика.
3. Вы осознаете главную ценность автотестов: они ловят регрессию (защищают то, что уже работало) и дают мгновенную обратную связь программистам, а не генерируют потоки новых дефектов.
4. Вы морально готовы к тому, что автотесты требуют постоянного рефакторинга и поддержки.

В следующей теме мы углубимся в то, как именно автотесты встраиваются в цикл разработки (SDLC) и почему Пирамида тестирования — это главный компас любого автоматизатора.

Sources:
1. [Which Test Cases Should You Not Automate in 2026](https://avoautomation.com/blog/test-cases-to-not-automate)
2. [Manual Testing vs Automated Testing (2026): Costs, CI/CD & AI Impact](https://www.getpanto.ai/blog/manual-testing-vs-automated-testing)
3. [Manual Testing vs Automated Testing: Which One Do You Need?](https://muuktest.com/blog/manual-testing-vs-automation-testing)
4. [Manual vs Automated Testing: Which Wins?](https://www.ranger.net/post/manual-vs-automated-testing-which-wins)
5. [Automated vs Manual Testing: Pros, Cons, and the AI Shift](https://www.virtuosoqa.com/post/automated-vs-manual-testing)
6. [Manual Testing vs Automation Which One Does Your Product Actually Need?](https://www.testaces.com/manual-vs-automation-testing-guide.php)
7. [Software Testing Best Practices for 2026](https://bugbug.io/blog/test-automation/software-testing-best-practices/)
8. [Best practices for automation testing in 2026](https://www.deviqa.com/blog/the-cost-of-ignoring-best-practices-in-test-automation/)
9. [What is meant by "Automated tests don't find new bugs"?](https://sqa.stackexchange.com/questions/381/what-is-meant-by-automated-tests-dont-find-new-bugs/721)
10. [10 Quality Myths Busted](https://testrigor.com/blog/10-quality-myths-busted/)
11. [Test Management Best Practices and Tools for 2026: The Complete Guide](https://testomat.io/blog/test-management-best-practices-and-tools/)
12. [Test Automation Myths: Where the Truth Ends and the Myth Begins?](https://blog.qatestlab.com/2024/04/11/test-automation-myths-where-the-truth-ends-and-the-myth-begins/)
13. [A false myth: automated tests don’t uncover bugs](https://imalittletester.com/2017/03/14/a-false-myth-automated-tests-dont-uncover-bugs/)
