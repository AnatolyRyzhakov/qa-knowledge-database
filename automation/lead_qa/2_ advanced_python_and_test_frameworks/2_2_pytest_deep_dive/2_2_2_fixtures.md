# 📂 Module 2.2.2. Фикстуры: уровни видимости (scope), yield, фабрики фикстур

Привет! Мы добрались до сердца PyTest — **Фикстур (Fixtures)**. Это реализация паттерна Dependency Injection (Внедрение зависимостей) для автотестов.

На уровне джуна фикстура — это просто функция, которая возвращает тестовые данные. На уровне Senior/Lead — это инструмент управления жизненным циклом ресурсов, распределения памяти между воркерами (при параллельном запуске) и генерации сложных графов объектов. 

В 2025–2026 годах мы работаем с асинхронностью, микросервисами и распределенными прогонами. Давай разберем, как писать фикстуры, которые не сломают инфраструктуру.

---

## 🔭 1. Уровни видимости (Scopes) и ловушка `pytest-xdist`

У фикстур есть 5 уровней видимости (scopes):
1. `function` (по умолчанию) — вызывается заново для каждого теста.
2. `class` — один раз для класса.
3. `module` — один раз для `.py` файла.
4. `package` — один раз для папки с тестами.
5. `session` — один раз на весь запуск `pytest`.

### 🛑 Архитектурная ловушка: `session` scope != Глобальный Singleton
Самая частая ошибка, на которой "сыпятся" мидлы на собеседованиях. Ты пишешь фикстуру `scope="session"`, которая очищает базу данных или поднимает Docker-контейнер. Локально всё работает. Но в CI/CD ты запускаешь тесты в 4 потока через `pytest-xdist` (`pytest -n 4`), и внезапно база очищается 4 раза, а тесты падают с конфликтами портов.

**Почему?** `pytest-xdist` использует мультипроцессинг (Multiprocessing). Каждый воркер (процесс) — это отдельная независимая "сессия" PyTest. Следовательно, `scope="session"` выполнится **по одному разу в каждом процессе**.

**Как это решает Лид (Modern Way 2025-2026):**
Для того чтобы `session` фикстура выполнилась строго один раз на весь физический прогон (например, создание одного глобального токена авторизации для всех воркеров), мы должны использовать **Lock-файлы (FileLocks)** или современные плагины типа `pytest-shared-session-scope`.

```python
import pytest
import json
from filelock import FileLock

@pytest.fixture(scope="session")
def global_token(tmp_path_factory, worker_id):
    """Генерация токена 1 раз на весь xdist-прогон"""
    
    if worker_id == "master":
        # Если запускаем без xdist
        return generate_expensive_token()
        
    # Путь к общему файлу для всех воркеров
    root_tmp_dir = tmp_path_factory.getbasetemp().parent
    fn = root_tmp_dir / "data.json"
    
    # Блокировка: первый воркер генерирует токен, остальные ждут и просто читают его
    with FileLock(str(fn) + ".lock"):
        if fn.is_file():
            return json.loads(fn.read_text())
            
        token = generate_expensive_token()
        fn.write_text(json.dumps(token))
        return token
```

---

## ♻️ 2. Жизненный цикл (Setup и Teardown) через `yield`

Старый подход из `unittest` (`setUp` / `tearDown`) мертв. Современный PyTest использует **генераторы** (ключевое слово `yield`) для управления жизненным циклом. 

Код до `yield` — это Setup (подготовка). Сам `yield` отдает данные в тест. Код после `yield` — это Teardown (очистка). Очистка гарантированно выполнится, даже если внутри теста упал `assert`.

### ⚡ Тренд 2026: Async Yield Фикстуры
Поскольку мы перешли на `httpx` и `asyncio`, нам нужны асинхронные фикстуры. В современном `pytest-asyncio` для этого больше не нужны "танцы с бубном" и хаки — достаточно использовать декоратор `@pytest_asyncio.fixture` и `async def` вместе с `yield`.

```python
import pytest_asyncio
import httpx

@pytest_asyncio.fixture(scope="function")
async def async_api_client():
    # --- SETUP ---
    client = httpx.AsyncClient(base_url="https://api.test.com")
    print("\n[+] Async client opened")
    
    # --- YIELD (отдаем в тест) ---
    yield client
    
    # --- TEARDOWN ---
    await client.aclose()
    print("\n[-] Async client closed")
```

---

## 🏭 3. Фабрики Фикстур (Fixture Factories)

**Проблема:** Классическая фикстура не может принимать аргументы из теста. Что делать, если в одном тесте тебе нужен `User(role="admin")`, а в другом `User(role="guest")`? 

**Плохое решение:** Использовать `pytest.mark.parametrize("user", indirect=True)`. Это громоздко и убивает читаемость.
**Хорошее решение:** Использовать паттерн **Fixture Factory**.

Фикстура возвращает не объект, а **функцию (callable)**, которая этот объект создает. Это позволяет передавать любые аргументы прямо из тела теста!

### 💻 Пример: Паттерн Fixture Factory (Level: Senior)

```python
import pytest
from typing import Callable

# Фикстура возвращает вложенную функцию-замыкание
@pytest.fixture
def make_user() -> Callable:
    created_users =[] # Локальное состояние фикстуры для очистки БД

    # Это функция-фабрика, которую получит тест
    def _make_user(role: str = "guest", is_active: bool = True) -> dict:
        user_data = {"id": len(created_users)+1, "role": role, "active": is_active}
        
        # Эмуляция: INSERT INTO users ...
        print(f"В БД создан юзер: {user_data}")
        created_users.append(user_data)
        
        return user_data

    # Отдаем ФУНКЦИЮ в тест
    yield _make_user
    
    # Teardown: Удаляем ВСЕХ юзеров, созданных фабрикой во время этого теста
    for u in created_users:
        print(f"Удаление из БД юзера {u['id']}")


# --- Использование в тесте (максимально чисто и читаемо) ---
def test_admin_can_delete_guest(make_user):
    # Тест сам решает, с какими параметрами ему нужны юзеры!
    admin = make_user(role="admin")
    guest = make_user(role="guest", is_active=False)
    
    assert admin["role"] == "admin"
```
*Почему это круто:* Ты избавляешься от десятков однотипных фикстур (`admin_user`, `guest_user`, `banned_user`). У тебя есть одна мощная Фабрика с настройкой параметров по умолчанию (Sensible Defaults).

---

## 🎤 Как это спросят на собеседовании

> **Вопрос 1: "Почему фикстура со `scope='session'` выполняется 4 раза, если я запускаю тесты через `pytest -n 4`?"**
* **Ответ Лида:** "Потому что `pytest-xdist` создает 4 независимых worker-процесса. 'Session' в PyTest означает жизненный цикл одного процесса (инстанса раннера). Воркеры изолированы друг от друга и не делят общую память (Global Interpreter Lock). Чтобы запустить фикстуру реально один раз на всю ферму воркеров (например, для поднятия БД), я должен использовать синхронизацию между процессами — обычно это FileLock, где первый воркер выполняет 'тяжелую' работу, сохраняет результат в кэш-файл, а остальные 3 воркера просто читают его из файла".

> **Вопрос 2: "Чем отличается `return` от `yield` внутри фикстуры?"**
* **Ответ Лида:** "Если я использую `return`, фикстура просто отработает, отдаст значение и завершится. Я не смогу выполнить код очистки ресурсов (Teardown). Если я использую `yield`, выполнение фикстуры ставится на паузу. PyTest передает сгенерированное значение в тест, запускает тест, а после его завершения (неважно, успешен он или упал с исключением) возвращается в фикстуру и довыполняет код, написанный после `yield`. Это замена старому подходу с `request.addfinalizer()`".

> **Вопрос 3: "Мне нужна фикстура, которая генерирует JSON-payload для создания заказа. Но в каждом тесте мне нужно менять только одно поле (например, `total_price`). Как ты это реализуешь?"**
* **Ответ Лида:** "Писать 10 разных фикстур — это антипаттерн. Использовать `indirect` параметризацию — нечитаемо. Я использую паттерн **Fixture Factory** (Фабрика фикстур). Фикстура `make_order_payload` будет возвращать внутри себя функцию `_builder(**kwargs)`. Внутри этой функции я задам дефолтный словарь с валидными данными, обновлю его через переданные `kwargs` и верну в тест. В самом тесте я напишу: `payload = make_order_payload(total_price=999)`".

---
Sources:
1. [pytest-shared-session-scope 0.5.2](https://pypi.org/project/pytest-shared-session-scope/)
2. [Run a pytest fixture only once when running with xdist](https://www.katzien.de/en/posts/2025-11-11-run-fixture-only-once-in-pytest-with-xdist/)
3. [pytest fixtures and threads synchronizations](https://stackoverflow.com/questions/54987936/pytest-fixtures-and-threads-synchronizations)
4. [pytest-xdist and session-scoped fixtures](https://developer.paylogic.com/articles/pytest-xdist-and-session-scoped-fixtures.html)
5. [pytest-shared-session-scope - pytest session scoped fixture that Just Works™ with xdist](https://www.reddit.com/r/Python/comments/1f3icze/pytestsharedsessionscope_pytest_session_scoped/?rdt=55555)
6. [Explicit is Better Than Implicit: Mastering Pytest Fixtures and Async Testing](https://www.ctrix.pro/blog/pytest-fixtures-async-testing-guide)
7. [How to make an async setup and teardown fixture in Pytest](https://stackoverflow.com/questions/79760973/how-to-make-an-async-setup-and-teardown-fixture-in-pytest)
8. [Pytest Fixture Factories: The Secret to Writing Flexible, Maintainable Tests](https://python.plainenglish.io/pytest-fixture-factories-the-secret-to-writing-flexible-maintainable-tests-2de79cf4084b)
