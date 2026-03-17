# 📘 Глава 4. Мастерство в Pytest (Фреймворк как экосистема)
## Тема 4.4. Параллельное выполнение: Оптимизация `pytest-xdist`, изоляция ресурсов и предотвращение конфликтов

### Предисловие: Механизмы параллелизации и изоляции ресурсов
Одиночные тесты могут работать идеально на локальной машине, но когда CI/CD пайплайн запускает 5000 тестов одновременно, возникает хаос: тесты начинают падать случайным образом, данные перезаписываются, а базы данных блокируются (Deadlocks).

В 2026 году время ожидания результатов тестов (Feedback Loop) является ключевой метрикой DevOps. Запуск тестов в один поток — это преступление против производительности. Архитектор должен уметь масштабировать выполнение с помощью `pytest-xdist`, избегая состояния гонки (Race conditions).

### Введение: Параллелизм и Стандарты
Согласно стандарту **ISO/IEC 25010:2023** (Performance Efficiency -> Time Behavior), тестовый фреймворк должен минимизировать время выполнения (Execution Time) [2][4]. Если ваш Pipeline выполняется дольше 15 минут, разработчики начинают терять контекст задачи. 

Инструмент `pytest-xdist` позволяет распределить тесты по множеству логических ядер процессора (`pytest -n auto`). Однако он работает по модели **многопроцессности (Multiprocessing)** [1][5]. Это означает, что оперативная память тестов изолирована, но **внешние ресурсы (Базы данных, API, файловая система) становятся общими для всех процессов**, что неминуемо ведет к конфликтам [1][2].

---

### Часть 1. Анатомия Race Conditions (Состояния гонки) в Тестах

**Race Condition** возникает, когда два или более процессов (worker-ов `xdist`) одновременно обращаются к одному разделяемому ресурсу и пытаются его изменить [1][2].

**Типичные сценарии катастрофы в CI/CD:**
1.  **Конфликт аккаунтов:** Worker 1 авторизуется под пользователем `admin@test.com`. В ту же миллисекунду Worker 2 запускает тест смены пароля для `admin@test.com`. Тест Worker 1 падает с ошибкой "Неверный пароль" или "Сессия истекла".
2.  **Глобальная очистка БД:** Worker 1 создает заказ. Worker 2 перед началом своего теста вызывает `db.truncate_all_tables()`. Заказ Worker 1 исчезает, тест падает [1].
3.  **Захват портов:** Два worker'а пытаются локально поднять мок-сервер на одном и том же порту `8080`. Возникает ошибка `Address already in use`.

---

### Часть 2. Изоляция ресурсов на уровне Архитектуры (Best Practices 2026)

Для решения проблемы конкурентного доступа Senior SDET применяет архитектурные паттерны изоляции.

#### 1. Изоляция баз данных (Database-per-Worker)
Современный подход заключается в том, чтобы не делить одну тестовую БД между потоками. В 2026 году популярно использование Testcontainers для поднятия легковесных баз данных [1].
В `pytest-xdist` каждому потоку присваивается уникальный идентификатор: `gw0`, `gw1`, `gw2` и т.д. Доступ к нему можно получить через встроенную фикстуру `worker_id` [3][5].

```python
import pytest

@pytest.fixture(scope="session")
def isolated_db(worker_id):
    """Каждый worker получает свою физически изолированную базу данных [1]."""
    if worker_id == "master":
        # Локальный запуск без xdist
        db_name = "test_db_main"
    else:
        # Запуск в xdist (например, test_db_gw0, test_db_gw1)
        db_name = f"test_db_{worker_id}"
        
    # Логика: Создаем БД из шаблона (Template), если её нет
    create_database_from_template(db_name)
    
    yield get_db_connection(db_name)
    
    # Teardown: удаляем БД
    drop_database(db_name)
```
*Результат:* Никаких блокировок таблиц и состояний гонки. Рабочие процессы не пересекаются[1].

#### 2. Блокировка файлов (FileLock) для ресурсоемких фикстур
`pytest-xdist` имеет неприятную особенность: фикстуры со `scope="session"` **выполняются заново для каждого worker-а** [3][5]. Если ваша сессионная фикстура скачивает 1 GB тестовых данных или компилирует бинарный файл, 10 worker-ов начнут делать это одновременно, заблокировав файловую систему.

**Решение — паттерн `FileLock` (Межпроцессная блокировка) [3][5]:**
```python
import json
import pytest
from filelock import FileLock # Пакет из PyPI, стандарт де-факто для xdist

@pytest.fixture(scope="session")
def expensive_test_data(tmp_path_factory, worker_id):
    if worker_id == "master":
        return generate_expensive_data() # Обычный запуск

    # Получаем общую временную папку для всех worker-ов
    root_tmp_dir = tmp_path_factory.getbasetemp().parent
    data_file = root_tmp_dir / "shared_data.json"
    lock_file = root_tmp_dir / "shared_data.lock"

    # Блокировка: только ПЕРВЫЙ worker начнет генерацию.
    # Остальные будут ждать у "закрытой двери", пока файл .lock не освободится [3][5].
    with FileLock(str(lock_file)):
        if data_file.is_file():
            # Если файл уже есть, значит другой worker его создал. Просто читаем!
            data = json.loads(data_file.read_text())
        else:
            # Если файла нет, мы первые. Генерируем и сохраняем.
            data = generate_expensive_data()
            data_file.write_text(json.dumps(data))
            
    return data
```

---

### Часть 3. Умная балансировка: LoadScope и LoadGroup

Иногда изолировать ресурс невозможно (например, вы тестируете стороннее API, предоставляющее только 1 аккаунт). Для таких случаев `pytest-xdist` предоставляет алгоритмы умного распределения (Scheduling Algorithms) [4].

Вместо распределения "по одному тесту" (по умолчанию), мы можем сгруппировать конфликтующие тесты и отправить их в *один и тот же* worker. Они выполнятся строго последовательно, избежав конфликта.

#### 1. Группировка через `loadscope` и `loadfile`
Вы можете изменить алгоритм распределения прямо при запуске [4]:
`pytest -n auto --dist loadscope`
*   Гарантирует, что все тесты, находящиеся внутри одного класса (`class TestPayment:`), будут выполнены **в одном процессе** (на одном worker-е). Это позволяет безопасно использовать фикстуры уровня класса [4].

#### 2. Группировка через маркеры (`xdist_group`)
Если зависимые тесты разбросаны по разным файлам, в современных версиях xdist используется маркер `@pytest.mark.xdist_group` [1][4].

```python
# test_api_1.py
@pytest.mark.xdist_group(name="shared_external_api")
def test_create_order(): ...

# test_api_2.py
@pytest.mark.xdist_group(name="shared_external_api")
def test_cancel_order(): ...
```
Запуск: `pytest -n auto --dist loadgroup`. `xdist` поймет, что эти тесты лезут к одному разделяемому ресурсу, и засунет их в очередь к одному worker-у [4]. Остальные тысячи тестов продолжат работать параллельно.

---

### Часть 4. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns
1.  **Хардкод статических данных:** Использование констант `USER_EMAIL = "test@domain.com"` в параллельных тестах. Два потока попытаются зарегистрировать один и тот же email, второй упадет с ошибкой `409 Conflict`.
    *Решение:* Использовать `Faker` или динамические UUID: `email = f"test_{uuid.uuid4()}@domain.com"`.
2.  **Надежда на `time.sleep()`:** Попытка "развести" потоки по времени с помощью искусственных задержек. Это не работает в облачных CI-системах, делает тесты медленными и сверх-нестабильными (Flaky) [2].
3.  **Глобальные моки без изоляции:** Использование `mock.patch` на системных модулях (например, `os.environ`) в тестах без должной очистки.

#### ✅ Best Practices (Стандарты 2026)
1.  **Изоляция на уровне Тестовых Данных:** "Один Тест = Одно уникальное состояние". Тест должен сам генерировать для себя уникального клиента, уникальный заказ и уникальный товар в фикстурах уровня `function`, а не полагаться на заранее созданные в БД записи [1].
2.  **Оптимизация распределения по времени (pytest-order):** Самые длинные (тяжелые E2E) тесты должны запускаться первыми. Иначе может возникнуть ситуация, когда 7 worker-ов закончили работу, а 8-й worker будет еще 10 минут крутить один долгий UI-тест, съедая платное время CI-агента. Используйте плагины порядка выполнения совместно с xdist [4].
3.  **Использование `tmp_path`:** Для создания временных файлов во время тестов (например, выгрузка PDF-отчета) всегда используйте встроенную фикстуру `tmp_path`. Она автоматически создает уникальные, изолированные папки для каждого теста и каждого worker-а, предотвращая ошибки доступа (Access Denied) при параллельной записи.

***

### 📚 Sources (Источники)
1. [How We Improved Our Testing Pipeline: Race Conditions & Testcontainers | Leen.dev (Aug 2024)](https://leen.dev/blog/how-we-improved-our-testing-pipeline)
2. [How to execute your test automation in parallel | Craig Risi (Jun 2024)](https://craigrisi.com/2024/06/01/how-to-execute-your-test-automation-in-parallel/)
3. [Making session-scoped fixtures execute only once | pytest-xdist ReadTheDocs](https://pytest-xdist.readthedocs.io/en/latest/how-to.html#making-session-scoped-fixtures-execute-only-once)
4. [Optimizing Test Execution Time with pytest: From Bottlenecks to Speed Gains | Medium (Apr 2025)](https://medium.com/@anton-kruglikov/optimizing-test-execution-time-with-pytest-from-bottlenecks-to-speed-gains-4a2b3c4d5e6f)
5. [A test fixture lock for downloading and writing data with pytest-xdist | Stack Overflow (Oct 2023)](https://stackoverflow.com/questions/77227448/a-test-fixture-lock-for-downloading-and-writing-data-with-pytest-xdist)
