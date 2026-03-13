# Блок 3: Инфраструктура и Аналитика (Жизнь тестов на сервере)
## Тема 7: Как автотесты запускаются в реальной жизни (Основы CI/CD)

Представьте ситуацию: вы написали отличный автотест, запустили его у себя на ноутбуке, он прошел успешно, и вы с чистой совестью ушли домой. Ночью разработчик внес изменения в код продукта и сломал логику авторизации. Утром сломанный продукт ушел к пользователям, потому что **ваш автотест спал вместе с вашим выключенным ноутбуком**.

В индустрии есть железное правило: **Тесты, которые запускаются только на машине автоматизатора — мертвы и не приносят пользы бизнесу**. Чтобы автотесты защищали продукт круглосуточно, их внедряют в процессы **CI/CD**. 

Давайте разберемся, что это такое и как устроено "под капотом" в 2026 году.

---

### 1. Что такое CI/CD простыми словами?

**CI/CD (Continuous Integration / Continuous Deployment)** — это конвейер (пайплайн) автоматизированной доставки кода от компьютера разработчика до серверов (Production), где продуктом пользуются реальные клиенты [[1](https://medium.com/@yuwanirashipaba/the-role-of-qa-in-ci-cd-pipelines-ensuring-quality-at-speed-e0b59fa0e631)]. 

В этом конвейере автотесты играют роль строгого «фейс-контроля» или таможни [[2](https://automationhq.ai/ci-cd-automated-testing-trends/)].

*   **CI (Непрерывная интеграция):** Практика, при которой разработчики по несколько раз в день сливают свой новый код в общую базу (репозиторий) [3]. Как только разработчик пытается добавить код, CI-сервер (например, GitLab CI, GitHub Actions или Jenkins) просыпается, скачивает этот код и **автоматически запускает ваши автотесты** [[1](https://medium.com/@yuwanirashipaba/the-role-of-qa-in-ci-cd-pipelines-ensuring-quality-at-speed-e0b59fa0e631)][[2](https://automationhq.ai/ci-cd-automated-testing-trends/)].
    *   *Если тесты зеленые (Passed):* Код признается безопасным и пропускается дальше.
    *   *Если тесты красные (Failed):* CI-сервер блокирует код, бьет тревогу и не дает разработчику сломать продукт [[4](https://www.baserock.ai/blog/ci-cd-integration-testing-best-practices-tools)].
*   **CD (Непрерывная доставка/развертывание):** Если этап CI (включая ваши тесты) пройден успешно, автоматика сама доставляет этот проверенный код на тестовые стенды или сразу боевые серверы к пользователям [[1](https://medium.com/@yuwanirashipaba/the-role-of-qa-in-ci-cd-pipelines-ensuring-quality-at-speed-e0b59fa0e631)]. 

---

### 2. Триггеры: Когда именно запускаются тесты?

Вам не нужно каждый раз просить сервер: *"Эй, запусти мои тесты"*. В современных пайплайнах 2026 года всё настроено на **триггеры** (события, запускающие процесс) [[2](https://automationhq.ai/ci-cd-automated-testing-trends/)].

1.  **По событию Merge Request / Pull Request (Самый важный триггер):**
    Разработчик закончил делать фичу и просит влить её в главную ветку. В этот момент запускаются **Unit-тесты** и быстрые **API-тесты** [[1](https://medium.com/@yuwanirashipaba/the-role-of-qa-in-ci-cd-pipelines-ensuring-quality-at-speed-e0b59fa0e631)]. Это концепция *Shift-Left*, о которой мы говорили в Блоке 1: поймать баг до того, как код вообще попадет в основной проект [[4](https://www.baserock.ai/blog/ci-cd-integration-testing-best-practices-tools)][[5](https://www.quales.tech/articles/the-future-of-qa-in-agile-and-devOps-why-traditional-testing-wont-survive)].
2.  **По расписанию (Nightly Builds / Ночные прогоны):**
    Тяжелые UI-тесты через браузер могут идти долго (30–60 минут). Разработчик не может ждать столько времени при каждом сохранении кода. Поэтому E2E-тесты запускаются ночью [[1](https://medium.com/@yuwanirashipaba/the-role-of-qa-in-ci-cd-pipelines-ensuring-quality-at-speed-e0b59fa0e631)][[6](https://mohamedsaidibrahim.medium.com/jenkins-ci-cd-for-test-automation-a-complete-guide-for-qa-engineers-73dff2ae2f3f)]. Утром команда открывает отчет (например, Allure) и видит, не сломалось ли чего за вчерашний день.
3.  **По кнопке (Manual Dispatch / On-demand):**
    Иногда QA-инженеру нужно проверить конкретную часть системы перед важным релизом. Вы заходите в веб-интерфейс CI-системы, выбираете нужный набор тестов (например, `@Smoke`) и нажимаете кнопку "Run".

---

### 3. Где и как запускаются браузеры? (Headless и Docker)

Здесь у новичков часто возникает ступор: *"Если тест кликает по кнопкам на экране, значит, на сервере CI/CD должен стоять монитор, где двигается курсор мышки?"*

Конечно же, нет. Серверы — это просто мощные компьютеры в дата-центрах, у них нет экранов. Автотесты работают там благодаря двум технологиям:

*   **Headless-режим (Безголовые браузеры):**
    В 2026 году все современные фреймворки (Playwright, Cypress, Selenium) запускают браузеры (Chrome, Firefox) в специальном "безголовом" режиме [[7](https://www.oreateai.com/blog/methods-and-practices-for-running-headless-browsers-in-docker/a7a724aa40cd1fa7c30d1fbaef049b3a)][[8](https://applitools.com/blog/headless-mode-for-tests-webdriverio/)]. Браузер полноценно работает, рендерит HTML, выполняет JavaScript и кликает по кнопкам в оперативной памяти сервера, но **не отрисовывает графический интерфейс (окно)** [[7](https://www.oreateai.com/blog/methods-and-practices-for-running-headless-browsers-in-docker/a7a724aa40cd1fa7c30d1fbaef049b3a)][[8](https://applitools.com/blog/headless-mode-for-tests-webdriverio/)]. Это экономит ресурсы и позволяет тестам работать в 2–3 раза быстрее [[8](https://applitools.com/blog/headless-mode-for-tests-webdriverio/)][[9](https://www.baeldung.com/ops/docker-google-chrome-headless)].
*   **Контейнеризация (Docker):**
    Знаменитая проблема *"А на моем компьютере всё работало!"* решается Докером [[7](https://www.oreateai.com/blog/methods-and-practices-for-running-headless-browsers-in-docker/a7a724aa40cd1fa7c30d1fbaef049b3a)][[9](https://www.baeldung.com/ops/docker-google-chrome-headless)]. Docker упаковывает ваши тесты, нужную версию Node.js/Python, сам Headless-браузер и все необходимые системные шрифты в изолированную "коробку" (контейнер) [[7](https://www.oreateai.com/blog/methods-and-practices-for-running-headless-browsers-in-docker/a7a724aa40cd1fa7c30d1fbaef049b3a)][[9](https://www.baeldung.com/ops/docker-google-chrome-headless)]. Этот контейнер одинаково успешно и предсказуемо запустится и на вашем Mac, и на Windows у разработчика, и на Linux-сервере в CI/CD пайплайне [[7](https://www.oreateai.com/blog/methods-and-practices-for-running-headless-browsers-in-docker/a7a724aa40cd1fa7c30d1fbaef049b3a)][[9](https://www.baeldung.com/ops/docker-google-chrome-headless)].

---

### 4. Роль Junior AQA в процессе CI/CD

В 2026 году от Junior-специалиста **не требуют** умения с нуля поднимать серверы Jenkins или писать сложные DevOps-скрипты маршрутизации [[5](https://www.quales.tech/articles/the-future-of-qa-in-agile-and-devOps-why-traditional-testing-wont-survive)][[6](https://mohamedsaidibrahim.medium.com/jenkins-ci-cd-for-test-automation-a-complete-guide-for-qa-engineers-73dff2ae2f3f)]. Это задача DevOps-инженеров или Senior AQA.

Однако вы обязаны уметь **жить внутри этого конвейера**:
1.  **Чтение логов CI-сервера:** Если конвейер остановился (упал), вы должны уметь открыть вкладку Console в GitHub Actions или GitLab CI и прочитать лог ошибки, чтобы понять: упал тест из-за бага, или просто моргнула сеть [[1](https://medium.com/@yuwanirashipaba/the-role-of-qa-in-ci-cd-pipelines-ensuring-quality-at-speed-e0b59fa0e631)].
2.  **Работа с Артефактами (Artifacts):** Поскольку тесты идут в "безголовом" режиме на удаленном сервере, вы не видите глазами, что там происходило. Для этого фреймворки сохраняют "артефакты" — видеозаписи прогона, скриншоты в момент падения и HTML-отчеты [[6](https://mohamedsaidibrahim.medium.com/jenkins-ci-cd-for-test-automation-a-complete-guide-for-qa-engineers-73dff2ae2f3f)]. Ваша задача — скачать их из CI и прикрепить к баг-репорту в Jira.
3.  **Культура "Stop the Line" (Останови конвейер):** В CI/CD есть правило: если главная ветка красная (тесты упали), вся команда бросает свои текущие дела и чинит проект. Если ваши автотесты регулярно падают из-за того, что они плохо написаны (flaky), вы будете останавливать работу десятков разработчиков. Именно поэтому нестабильные тесты нужно оперативно чинить или отключать (скипать).

---

### Резюме для самопроверки

1.  **CI/CD** — это автоматизированный конвейер. **CI** отвечает за проверку нового кода автотестами при каждом коммите, а **CD** — за его доставку на сервер [[1](https://medium.com/@yuwanirashipaba/the-role-of-qa-in-ci-cd-pipelines-ensuring-quality-at-speed-e0b59fa0e631)][[3](https://www.compunnel.com/blogs/continuous-integration-and-continuous-deployment-ci-cd-in-quality-assurance-qa/)].
2.  Автотесты запускаются **автоматически (триггеры)**: при слиянии кода разработчиком (быстрые API/Unit проверки) или по ночному расписанию (долгие UI тесты) [[1](https://medium.com/@yuwanirashipaba/the-role-of-qa-in-ci-cd-pipelines-ensuring-quality-at-speed-e0b59fa0e631)].
3.  На сервере тесты работают в **Headless-режиме** — браузер функционирует полноценно, но без отрисовки визуального окна (GUI) [[8](https://applitools.com/blog/headless-mode-for-tests-webdriverio/)][[10](https://www.nimbleway.com/blog/headless-browser-testing-guide)].
4.  Для стабильности и изоляции окружения используется **Docker** — технология, позволяющая запускать тесты в абсолютно одинаковых условиях везде [[7](https://www.oreateai.com/blog/methods-and-practices-for-running-headless-browsers-in-docker/a7a724aa40cd1fa7c30d1fbaef049b3a)][[9](https://www.baeldung.com/ops/docker-google-chrome-headless)].
5.  Роль Junior AQA — анализировать результаты пайплайнов, изучать **артефакты** (логи, видео) упавших тестов и не допускать нестабильных проверок в конвейере [[1](https://medium.com/@yuwanirashipaba/the-role-of-qa-in-ci-cd-pipelines-ensuring-quality-at-speed-e0b59fa0e631)][[6](https://mohamedsaidibrahim.medium.com/jenkins-ci-cd-for-test-automation-a-complete-guide-for-qa-engineers-73dff2ae2f3f)].

Теперь вы знаете, как ваши тесты работают на "боевом дежурстве". Но что делать, если конвейер загорелся красным? В следующей теме мы научимся **анализировать результаты, читать отчеты и бороться с плавающими тестами (Flaky tests)**!

Sources
1. [The Role of QA in CI/CD Pipelines: Ensuring Quality at Speed](https://medium.com/@yuwanirashipaba/the-role-of-qa-in-ci-cd-pipelines-ensuring-quality-at-speed-e0b59fa0e631)
2. [Continuous Integration/Continuous Delivery and the Future of Automated Testing: Trends to Watch](https://automationhq.ai/ci-cd-automated-testing-trends/)
3. [Continuous Integration and Continuous Deployment (CI/CD) in Quality Assurance (QA)](https://www.compunnel.com/blogs/continuous-integration-and-continuous-deployment-ci-cd-in-quality-assurance-qa/)
4. [CI/CD Integration Testing: Best Practices & Tools](https://www.baserock.ai/blog/ci-cd-integration-testing-best-practices-tools)
5. [The Future of QA in Agile and DevOps: Why Traditional Testing Won’t Survive](https://www.quales.tech/articles/the-future-of-qa-in-agile-and-devOps-why-traditional-testing-wont-survive)
6. [Jenkins & CI/CD for Test Automation: A Complete Guide for QA Engineers](https://mohamedsaidibrahim.medium.com/jenkins-ci-cd-for-test-automation-a-complete-guide-for-qa-engineers-73dff2ae2f3f)
7. [Methods and Practices for Running Headless Browsers in Docker](https://www.oreateai.com/blog/methods-and-practices-for-running-headless-browsers-in-docker/a7a724aa40cd1fa7c30d1fbaef049b3a)
8. [Using Headless Mode for Your Tests – When, Why and How (demonstration with WebdriverIO)](https://applitools.com/blog/headless-mode-for-tests-webdriverio/)
9. [Run Google Chrome headless in Docker](https://www.baeldung.com/ops/docker-google-chrome-headless)
10. [What Is Headless Browser Testing? A Complete Guide for 2025](https://www.nimbleway.com/blog/headless-browser-testing-guide)
