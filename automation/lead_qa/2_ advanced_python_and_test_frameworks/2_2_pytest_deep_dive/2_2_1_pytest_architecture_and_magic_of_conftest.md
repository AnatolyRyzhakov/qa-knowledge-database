# 📂 Module 2.2.1. Архитектура PyTest и магия `conftest.py`

Привет! Мы переходим к **PyTest**. Для джуна PyTest — это просто библиотека, в которой есть слово `assert` и фикстуры. Для Лида / Архитектора PyTest — это мощнейший движок, построенный на хуках (hooks) и плагинной архитектуре.

Понимание того, как PyTest работает под капотом, — это твой ключ к написанию кастомных отчетов, перехвату падений тестов и интеграции с CI/CD.

---

## 🏗️ 1. Архитектура PyTest: `pluggy` и Хуки (Hooks)

PyTest не является монолитной программой. Его ядро — это крошечный движок, который управляется библиотекой **`pluggy`**. 
Всё, что ты видишь в PyTest (фикстуры, цветной вывод в консоль, сбор тестов), реализовано как плагины.

**Как работает `pluggy` (Hook-based architecture):**
В процессе работы PyTest проходит несколько стадий (Инициализация -> Сбор тестов -> Запуск -> Отчет). На каждой стадии он "кричит" в систему: *"Я начинаю собирать тесты, кто хочет вмешаться?"* (вызывает хук `pytest_collection_modifyitems`). Любой плагин может перехватить этот хук и изменить логику.

**Магия `assert` (Assertion Rewriting):**
Почему в обычном Python `assert 1 == 2` выведет просто `AssertionError`, а в PyTest он покажет красивые подробности: `assert 1 == 2, where 1 is len([0])`?
Потому что PyTest перехватывает импорт твоих тестовых файлов и на лету **переписывает их AST-дерево (Abstract Syntax Tree)**, встраивая туда свой код для интроспекции переменных.

---

## 🎩 2. Что такое `conftest.py` на самом деле?

Многие думают, что `conftest.py` — это просто "файлик для фикстур". С точки зрения архитектуры, `conftest.py` — это **локальный плагин PyTest**.

**Правила магии `conftest.py`:**
1. **Auto-Discovery (Автообнаружение):** Тебе **НЕЛЬЗЯ** импортировать `conftest.py` в свои тесты (например, `from conftest import my_fixture`). PyTest найдет его сам и прокинет фикстуры через механизм Dependency Injection.
2. **Иерархия (Scoping):** Файл `conftest.py` действует только на ту директорию, в которой лежит, и на всех её "детей".
   * Фикстура из `tests/api/conftest.py` **не будет** доступна в `tests/ui/test_login.py`.
   * Фикстура из корневого `tests/conftest.py` доступна везде.

---

## 🛑 3. Антипаттерн "God Object" (Проблема разрастания)

На проектах, которые живут больше года, корневой `tests/conftest.py` часто превращается в "помойку" на 3000 строк кода. В нём смешиваются инициализация драйвера браузера, коннекты к БД, генерация данных и хуки отчетов.

В 2025–2026 годах это считается серьезным архитектурным запахом (Code Smell). Это усложняет Code Review и приводит к конфликтам при слиянии веток (Merge Conflicts).

---

## 🛠️ 4. Решение 2026 года: Модульный `conftest.py` (через `pytest_plugins`)

Вместо того чтобы хранить всё в одном файле, Senior SDET разбивает фикстуры на логические домены (модули) и импортирует их через встроенный механизм PyTest — переменную `pytest_plugins`.

### 💻 Как выглядит правильная архитектура:

**Структура директорий:**
```text
tests/
├── plugins/              # Папка для изолированных модулей фикстур
│   ├── api_fixtures.py   # Фикстуры для httpx
│   ├── db_fixtures.py    # Фикстуры базы данных
│   └── ui_fixtures.py    # Инициализация Playwright
├── conftest.py           # Корневой файл (теперь он почти пустой!)
└── test_users.py
```

**Файл `tests/plugins/db_fixtures.py` (Никаких импортов conftest!):**
```python
import pytest
from sqlalchemy import create_engine

@pytest.fixture(scope="session")
def db_connection():
    engine = create_engine("postgresql://user:pass@localhost/db")
    yield engine
    engine.dispose()
```

**Файл `tests/conftest.py` (Архитектурный шедевр):**
```python
# Вместо 3000 строк кода, мы оставляем только массив плагинов!
# PyTest сам пойдет в эти файлы, найдет там хуки и фикстуры и зарегистрирует их.

pytest_plugins =[
    "tests.plugins.api_fixtures",
    "tests.plugins.db_fixtures",
    "tests.plugins.ui_fixtures",
]

# Здесь оставляем ТОЛЬКО глобальные хуки конфигурации, например:
def pytest_configure(config):
    """Регистрация кастомных маркеров"""
    config.addinivalue_line("markers", "smoke: mark test as smoke")
```
*Результат:* Идеально чистый код, соблюдение принципа единой ответственности (SRP). Разработчики БД-тестов больше не трогают файл с фикстурами UI.

---

## 🎤 Как это спросят на собеседовании

> **Вопрос 1: "Почему нельзя просто написать `from tests.conftest import my_fixture` в файле с тестом?"**
* **Ответ Лида:** "PyTest использует паттерн Dependency Injection (Внедрение зависимостей). Когда PyTest собирает тесты, он анализирует сигнатуры тестовых функций (их аргументы) и сопоставляет их с реестром обнаруженных фикстур. Если мы сделаем прямой импорт, мы сломаем жизненный цикл PyTest: например, фикстура может потребовать выполнения хуков setup/teardown или учета scope (session/function), что при прямом импорте просто не сработает. `conftest.py` — это локальный плагин, а не обычный Python-модуль."

> **Вопрос 2: "Ты пришел на проект. Открываешь `conftest.py`, а там 5000 строк кода. Как будешь рефакторить, чтобы ничего не сломать?"**
* **Ответ Лида:** "Я применю подход разделения на плагины. Создам директорию `fixtures` или `plugins` внутри `tests/`. Разобью 5000 строк по смыслу: `auth_fixtures.py`, `browser_fixtures.py`, `clients.py`. Затем в корневом `conftest.py` удалю весь этот код и добавлю переменную `pytest_plugins = ['tests.fixtures.auth_fixtures', ...]`. Для PyTest и самих тестов абсолютно ничего не изменится — они даже не заметят рефакторинга, так как плагины будут загружены на этапе инициализации."

> **Вопрос 3: "Мне нужно, чтобы определенная фикстура (например, `create_admin`) была доступна только для API тестов, но не засоряла UI тесты. Как это сделать архитектурно?"**
* **Ответ Лида:** "Я использую иерархию (Scoping) файлов `conftest.py`. Я положу эту фикстуру не в корневой `tests/conftest.py`, а создам новый файл `tests/api/conftest.py`. По правилам PyTest, фикстуры из этого файла будут 'видны' только тестам внутри папки `tests/api/` и её подпапок. UI-тесты, лежащие в `tests/ui/`, об этой фикстуре даже не узнают."

---
Sources:
1. [pluggy](https://pluggy.readthedocs.io/en/stable/)
2. [Writing hook functions](https://docs.pytest.org/en/stable/how-to/writing_hook_functions.html)
3. [Writing plugins](https://docs.pytest.org/en/stable/how-to/writing_plugins.html)
4. [pytest Tutorial: Effective Python Testing](https://realpython.com/pytest-python-testing/)
5. [pytest fixtures: explicit, modular, scalable](https://docs.pytest.org/en/6.2.x/fixture.html)
6. [Mastering Pytest: The Complete Guide to Modern Python Testing](https://python.plainenglish.io/mastering-pytest-the-complete-guide-to-modern-python-testing-8073d2cc284c)
7. [Python Unit Tests](https://medium.com/@usmanmushtaq1990/python-unit-tests-9acd5ce0d5a8)
