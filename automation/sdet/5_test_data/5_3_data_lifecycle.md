# 📘 Глава 5. Управление тестовыми данными (Test Data Management)
## Тема 5.3. Жизненный цикл данных: Идемпотентность тестов и безопасный Teardown

### Предисловие
Даже если вы генерируете уникальные синтетические данные (Faker) или поднимаете изолированные контейнеры, отсутствие стратегии очистки (Teardown) быстро приведет к деградации тестовых сред.

В 2026 году архитекторы автоматизации фокусируются на концепции **Идемпотентности (Idempotency)**. Тест должен быть "невидимкой": он приходит, делает свою работу и уходит, не оставляя никаких следов в системе. 

### Введение: Контекст стандартов (ISO 25010 и ISTQB v3.0)
Согласно стандарту качества **ISO/IEC 25010:2023**, характеристика **Performance Efficiency** включает параметр **Resource Utilization** (Использование ресурсов) [4]. Если ваши тесты ежедневно создают тысячи пользователей, не удаляя их, база данных рано или поздно исчерпает лимит ID или место на диске, что приведет к падению всей системы (State Leakage) [5][6].
Обновленный стандарт **ISTQB Advanced Level Test Management v3.0** (вышедший в 2024–2025 гг.) требует внедрения стратегии "Data Lifecycle Management" (Управления жизненным циклом данных) еще на этапе планирования архитектуры автоматизации[8]. 

---

### Часть 1. Идемпотентность в контексте автотестов

В классическом программировании **идемпотентность** означает, что операция дает один и тот же результат независимо от того, сколько раз она была выполнена (например, HTTP-метод `PUT`) [3]. 

В автоматизации тестирования **Идемпотентный тест** — это тест, который полностью изолирован, не зависит от глобального состояния системы и **возвращает систему в исходное состояние после своего завершения** [1][2].
*   *"Я попытался переключить флаг в UI дважды, и тест прошел 50 раз, но на 51-й раз упал, потому что в БД закончились уникальные ID"*. Этот тест **не идемпотентен**, так как он загрязняет (contaminates) свое окружение при каждом прогоне[6].

**Архитектурное правило 2026 года:** 
Тест должен быть идемпотентным *независимо от того, прошел он успешно (Passed) или упал с ошибкой (Failed)*[1]. 

---

### Часть 2. Стратегии безопасного Teardown (Очистки)

Самая частая ошибка Junior-специалистов — размещение логики удаления данных в конце самого теста.
```python
# ❌ ПЛОХО: Если assert упадет, метод delete() никогда не выполнится. Данные "утекут" в БД.
def test_create_user(api_client):
    user_id = api_client.create_user("test@mail.com")
    assert api_client.get_user(user_id).status == "ACTIVE"
    api_client.delete_user(user_id) 
```

Senior SDET использует паттерны **Гарантированного Teardown**.

#### 1. Использование Фикстур с обработкой ошибок (Pytest `yield`)
Как мы обсуждали в Главе 4, код после `yield` в Pytest выполняется всегда. Но что, если *само* API удаления вернет ошибку (например, 500 Internal Server Error)? Teardown прервется, и фреймворк упадет [1].
В 2026 году стандартом является оборачивание логики Teardown в защитные блоки `try/except` [1][6].

```python
# ✅ ОТЛИЧНО: Безопасный Teardown с подавлением ошибок (Safe Cleanup)
@pytest.fixture
def temp_user(api_client, deterministic_faker):
    # 1. Setup
    user_id = api_client.create_user(deterministic_faker.email())
    
    # 2. Инжект в тест
    yield user_id 
    
    # 3. Safe Teardown
    try:
        response = api_client.delete_user(user_id)
        response.raise_for_status()
    except Exception as e:
        # Логируем сбой очистки, но НЕ роняем весь тестовый прогон!
        logger.error(f"Не удалось удалить пользователя {user_id}. Причина: {e}")
```

#### 2. Паттерн "Test Namespaces" (Пространства имен)
Для E2E тестирования, где один тест создает десятки связанных сущностей (Компания $\rightarrow$ Отдел $\rightarrow$ Сотрудник $\rightarrow$ Платеж), удалять их по одному слишком долго и ненадежно.
**Решение:** Перед тестом создается уникальный Tenant (Рабочее пространство / Namespace). Тест генерирует внутри него любой мусор. На этапе Teardown мы отправляем *один* API-запрос на удаление всего Tenant'а. БД сама каскадно удалит все вложенные сущности (Cascade Delete) [5].

---

### Часть 3. Что делать, если физическое удаление невозможно?

В Enterprise-системах (Fintech, ERP, Healthcare) часто запрещено использовать `DELETE` в базе данных из-за строгих аудиторских логов (Compliance) или внешних ключей (Foreign Keys). 

Если данные удалить физически нельзя, SDET применяет следующие паттерны:

1.  **Soft Deletion (Мягкое удаление) + Анонимизация:**
    На этапе Teardown автотест меняет статус сущности на `status="ARCHIVED"` и заменяет имя на случайное, чтобы освободить уникальный email/номер телефона для следующих прогонов.
2.  **Асинхронный Garbage Collection (Сборка мусора):**
    Вместо очистки данных в реальном времени (что замедляет CI/CD конвейер), каждый автотест при создании сущности присваивает ей специальный тег, например `created_by: "automation_script"`. 
    Отдельная DevOps-джоб (Cron job) запускается каждую ночь в 03:00 и жестко вычищает из тестовой базы все записи с этим тегом, используя права администратора БД (DBA). Это разгружает автотесты от логики удаления [5][7].

---

### Часть 4. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns
1.  **Использование общих данных (Shared State):** Запуск параллельных тестов, которые читают/модифицируют одного и того же "тестового клиента". Это гарантированный путь к состоянию гонки (Race conditions) и потере идемпотентности[3][7].
2.  **Отсутствие изоляции кэша:** Забывать очищать Redis или Memcached. Вы можете удалить пользователя из БД, но если его профиль остался в кэше, следующий тест может прочитать устаревшие данные (Stale Data). Teardown должен затрагивать все слои приложения [3].

#### ✅ Best Practices
1.  **Атомарность (Atomicity):** Проектируйте тесты так, чтобы они проверяли только одну бизнес-функцию и зависели от минимального количества данных [7].
2.  **Встроенные механизмы очистки HTTP-клиентов:** Если вы тестируете API микросервисов, используйте хуки (Lifecycle hooks) современных HTTP-клиентов для автоматической регистрации созданных ресурсов и их пакетного удаления в конце сессии [3].
3.  **Использование In-Memory баз данных:** Для интеграционных тестов (не E2E) создание временного `SQLite` хранилища в оперативной памяти для каждого теста — лучший способ гарантировать 100% идемпотентность. Как только тест завершен, память очищается ОС автоматически [1].

***

### 📚 Sources (Источники)
1. [8 Effective Strategies for Handling Flaky Tests - Codecov (Nov 2024)](https://about.codecov.io/blog/8-effective-strategies-for-handling-flaky-tests/)
2. [Best practices for creating end-to-end tests - Datadog (Jun 2020 / Updated 2025)](https://www.datadoghq.com/blog/end-to-end-testing-best-practices/)
3. [HTTP Client Testing Techniques for Integration - MoldStud (Oct 2025)](https://moldstud.com/articles/p-http-client-testing-techniques-for-integration-in-net-core)
4. [ISO/IEC 25010 Software Quality Model Overview - ISO25000.com](https://iso25000.com/index.php/en/iso-25000-standards/iso-25010)
5. [What is End to End Testing? Meaning, Examples, Use Cases - DevOpsSchool (Feb 2026)](https://www.devopsschool.org/blog/what-is-end-to-end-testing/)
6. [Every Programmer Should Know #1: Idempotency - Reddit Programming Community Discussions](https://www.reddit.com/r/programming/comments/16nxxkz/every_programmer_should_know_1_idempotency/)
7. [15 Common Automation Testing Pitfalls & How to Avoid Them Like a Pro - Medium (May 2025)](https://medium.com/@software-testing/15-common-automation-testing-pitfalls-how-to-avoid-them-like-a-pro-2c3f15c7a4b8)
8. [ISTQB CTAL-TM v3.0 (Test Management) Syllabus & Agile Integration (2024-2025 release)](https://www.istqb.guru/istqb-advanced-level-test-management-dumps-free-download/)
