# 📘 Глава 4. Мастерство в Pytest (Фреймворк как экосистема)
## Тема 4.2. Механизмы хуков (Custom Hooks): Кастомизация отчетов и пайплайнов через `conftest.py`

### Введение: Архитектура событий Pytest и Стандарты
Для Senior SDET понимание механизмов **Хуков (Hooks)** — это переход от написания тестов к созданию полноценных инструментов для CI/CD конвейера. 

В 2026 году бизнес требует от автоматизации не просто статуса "Pass/Fail", а мгновенной аналитики, оптимизации времени прогона (Green DevOps) и богатых отчетов с логами, скриншотами и видео [2]. Все это реализуется через `conftest.py` и систему хуков.

Согласно стандарту качества **ISO/IEC 25010**, характеристика **Maintainability** включает в себя **Analyzability** (Анализируемость) — насколько легко диагностировать причину сбоя системы. В рамках **ISTQB Advanced TAE** к фреймворкам предъявляется требование автоматического сбора диагностических данных (Test Logging) при падениях в CI/CD.

Pytest построен на **событийно-ориентированной архитектуре (Event-driven architecture)**. Его работа — это не монолитный скрипт, а строгий конвейер фаз: 

*Инициализация → Сбор тестов (Collection) → Выполнение (Run) → Отчетность (Reporting)* [4]. 

Хуки (Hooks) позволяют вам "вклиниться" в любую из этих фаз и изменить дефолтное поведение Pytest без модификации исходного кода самого фреймворка [1][4].

Все хуки традиционно размещаются в конфигурационном файле `conftest.py`.

---

### Часть 1. Кастомизация пайплайна: Сбор и сортировка тестов

Одной из главных задач Senior SDET в 2026 году является оптимизация CI/CD пайплайнов. Если у вас 10 000 тестов, запускать их все на каждый Pull Request слишком дорого и долго (тренд "Green DevOps" призывает экономить вычислительные мощности) [2].

Для динамического управления списком тестов используется хук `pytest_collection_modifyitems` [1][5].

**Пример 2026: Умная сортировка и фильтрация (Smart Execution)**
Вместо случайного порядка выполнения мы можем заставить Pytest всегда выполнять `Smoke`-тесты первыми (чтобы реализовать паттерн Fail-Fast) или динамически пропускать тесты, если модули продукта не менялись [2][5].

```python
# conftest.py
import pytest

def pytest_collection_modifyitems(config, items):
    """
    Хук вызывается после того, как Pytest собрал все тесты, но до их выполнения [5].
    """
    smoke_tests =[]
    regular_tests =[]
    
    # 1. Сортируем тесты: Smoke всегда идут первыми для стратегии Fail-Fast
    for item in items:
        if item.get_closest_marker("smoke"):
            smoke_tests.append(item)
        else:
            regular_tests.append(item)
            
    # Перезаписываем оригинальный список тестов отсортированным
    items[:] = smoke_tests + regular_tests

    # 2. Пример Green DevOps: Пропускаем тяжелые тесты, если передан спец. флаг [2]
    if config.getoption("--smart-skip"):
        skip_marker = pytest.mark.skip(reason="Пропущено умным фильтром CI/CD")
        for item in regular_tests:
            item.add_marker(skip_marker)
```

*(Для добавления кастомного флага `--smart-skip` в командную строку используется хук `pytest_addoption` [1]).*

---

### Часть 2. Кастомизация отчетов: Магия `pytest_runtest_makereport`

Самый востребованный хук на собеседованиях Senior SDET — это `pytest_runtest_makereport`. Он вызывается **трижды** для каждого теста (на этапах `setup`, `call` и `teardown`) [4]. 

Именно здесь реализуется привязка скриншотов, логов контейнера или сетевых дампов (HAR) к упавшим тестам в **Allure Report** или HTML-отчетах [3].

**Архитектурно правильная реализация автоматических скриншотов:**
Чтобы прикрепить скриншот, хуку нужен доступ к инстансу драйвера (браузера), который обычно живет внутри фикстуры теста.

```python
# conftest.py
import pytest
import allure

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """
    Хук-обертка для перехвата результатов выполнения теста [3][4].
    """
    # 1. Выполняем все остальные хуки и получаем результат (Report Object)
    outcome = yield
    rep = outcome.get_result()
    
    # Сохраняем результат выполнения в объект item (самого теста), 
    # чтобы другие фикстуры могли понять, упал тест или нет
    setattr(item, "rep_" + rep.when, rep)
    
    # 2. Ловим падения только на этапе самого теста (call), игнорируя setup/teardown
    if rep.when == "call" and rep.failed:
        # Динамически достаем фикстуру 'page' (например, из Playwright)
        if "page" in item.fixturenames:
            page = item.funcargs["page"]
            try:
                # Делаем скриншот и аттачим его в Allure Report [3]
                screenshot = page.screenshot(full_page=True)
                allure.attach(
                    screenshot, 
                    name=f"Screenshot_Failure_{item.name}", 
                    attachment_type=allure.attachment_type.PNG
                )
            except Exception as e:
                print(f"Ошибка при создании скриншота: {e}")
```

---

### Часть 3. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns (Как сломать CI/CD конвейер)
1.  **Отсутствие `try/except` внутри Хуков отчета:** Хук `pytest_runtest_makereport` вызывается в критический момент. Если браузер "умер" (Crash) во время теста, попытка сделать скриншот в хуке вызовет `WebDriverException` / `TargetClosedError`. Если эту ошибку не обернуть в `try/except`, упадет сам процесс генерации отчета, и в CI/CD вы не увидите ни логов, ни причины падения[3].
2.  **Загромождение корневого `conftest.py`:** Размещение сотен строк хуков для UI, API и БД в одном файле.
    *Решение:* Pytest позволяет создавать `conftest.py` в каждой директории (например, `tests/ui/conftest.py`). Хуки будут применяться только к тестам в этой папке и ее подпапках [1].
3.  **Изменение бизнес-состояния через Хуки:** Использование `pytest_runtest_setup` для авторизации в системе или очистки БД. Хуки предназначены для инфраструктуры (отчеты, сортировка, логирование). Бизнес-пред-условия должны находиться строго в **Фикстурах (Fixtures)**.

#### ✅ Best Practices (Стандарты 2026)
1.  **Использование `hookwrapper=True`:** В Pytest исторически хуки могут перезаписывать друг друга. Использование обертки (`yield`) гарантирует, что ваш код для отчета (Allure) отработает строго *после* того, как Pytest сформирует официальный статус теста (Pass/Fail) [1][3].
2.  **Интеграция с Observability (ELK/Datadog):** В современных компаниях результаты тестов отправляются не только в Allure, но и в системы мониторинга. Senior SDET использует хук `pytest_runtest_logreport(report)` для отправки метрик (имя теста, статус, длительность) через асинхронный HTTP-запрос прямо в Datadog/Elasticsearch для построения дашбордов стабильности релизов [4].
3.  **Использование `pytest_sessionfinish`:** Этот хук вызывается в самом конце прогона. Идеальное место для автоматической отправки уведомления в Slack/Teams с агрегированной статистикой (Passed: X, Failed: Y), а также для очистки "мусорных" Docker-контейнеров, которые могли "зависнуть" при падении пайплайна [1].

***

### 📚 Sources (Источники)
1. [Basic patterns and examples — pytest documentation (Good Integration Practices)](https://docs.pytest.org/en/latest/how-to/recipes.html)
2. [Green DevOps Practices: Your CI/CD Pipeline is Burning More Carbon Than Your Production Code | by The_Architect (Feb 2026)](https://levelup.gitconnected.com/green-devops-practices-your-ci-cd-pipeline-is-burning-more-carbon-than-your-production-code-19967c51dd2a)
3. [PyTest & Allure: Screenshot of failure in Allure report (Stack Overflow)](https://stackoverflow.com/questions/49573883/pytest-allure-screenshot-of-failure-in-allure-report)
4. [Pytest Documentation Overview | API Reference & Call Phases (Scribd)](https://www.scribd.com/document/561085202/pytest)
5. [How to Handle pytest Markers and `pytest_collection_modifyitems` (OneUptime, Feb 2026)](https://oneuptime.com/blog/pytest-markers/)
