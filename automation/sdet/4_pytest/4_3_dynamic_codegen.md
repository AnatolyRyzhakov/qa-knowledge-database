# 📘 Глава 4. Мастерство в Pytest (Фреймворк как экосистема)
## Тема 4.3. Динамическая кодогенерация: Параметризация на лету (из JSON/YAML) для White Label решений

### Предисловие: **Data-Driven Testing (DDT)** для масштабных проектов
В 2026 году многие SaaS-компании перешли на модель **White Label** (когда один и тот же продукт продается под разными брендами с разным дизайном, локализацией и набором фич). Писать отдельные тесты для каждого из 50 брендов-клиентов — это нарушение принципов DRY и ISO/IEC 25010 (Maintainability) [3]. Фреймворк должен содержать *один* набор тестов, который динамически подтягивает разные тестовые данные (JSON/YAML) в зависимости от того, какой бренд мы тестируем в данный момент.

### Введение: Почему базовый `@pytest.mark.parametrize` не подходит?
Каждый Junior-автоматизатор знает декоратор `@pytest.mark.parametrize`. Он отлично работает для простых проверок, когда данные захардкожены прямо в скрипте [3]. 

Но если вы попытаетесь прочитать большой JSON-файл прямо внутри декоратора (на уровне модуля), вы столкнетесь с проблемой: **этот код выполнится в момент импорта (Import Time)** [4]. Если файла нет, или его нужно динамически скачать из базы данных перед тестами, ваш фреймворк просто упадет (Crash) еще до начала тестовой сессии. 

В Enterprise-архитектурах чтение конфигураций должно происходить строго на этапе **сбора тестов (Collection Time)** [1].

---

### Часть 1. Архитектурный стандарт: Хук `pytest_generate_tests`

Для динамического создания тестов (кодогенерации) в Pytest используется встроенный хук `pytest_generate_tests(metafunc)`. Он позволяет проанализировать каждый тест перед запуском и динамически "размножить" его, подставив нужные данные из внешнего источника [1][4].

**Сценарий White Label:** У нас есть папка `configs/`, где лежат файлы `brand_A.json` и `brand_B.yaml`. В зависимости от переменной окружения `--brand`, Pytest должен сгенерировать тесты только для нужного клиента.

**Пример реализации в `conftest.py`:**

```python
import pytest
import json
import yaml
from pathlib import Path

# 1. Добавляем кастомный флаг CLI для выбора White Label бренда
def pytest_addoption(parser):
    parser.addoption("--brand", action="store", default="brand_A", help="Select White Label brand")

# 2. Хук кодогенерации
def pytest_generate_tests(metafunc):
    """
    Вызывается во время сбора тестов. Позволяет динамически внедрять параметры [1][4].
    """
    # Проверяем, ожидает ли функция аргумент 'brand_data'
    if "brand_data" in metafunc.fixturenames:
        brand_name = metafunc.config.getoption("--brand")
        file_path = Path(__file__).parent / f"configs/{brand_name}.json"
        
        # Защита: если конфига нет, пропускаем тест с понятным сообщением
        if not file_path.exists():
            pytest.skip(f"Конфигурация для бренда {brand_name} не найдена!")
            
        # Лениво читаем данные
        with open(file_path, "r", encoding="utf-8") as f:
            test_data = json.load(f) # Ожидаем массив тестовых сценариев
            
        # Динамически генерируем тесты!
        # Каждая запись в массиве JSON создаст отдельный, независимый Test Case.
        metafunc.parametrize(
            "brand_data", 
            test_data, 
            ids=[data.get("scenario_name", "unnamed") for data in test_data] # Читаемые имена в отчете
        )
```

**Использование в самом тесте (идеально чистый код):**
```python
# test_white_label.py
def test_checkout_flow(brand_data):
    # Тест не знает, откуда взялись данные. Он просто тестирует бизнес-логику.
    assert brand_data["currency"] in ["USD", "EUR"]
    assert brand_data["feature_toggles"]["enable_crypto_pay"] is not None
```

---

### Часть 2. Современная экосистема (2025–2026): Плагины и Валидация

Senior SDET не всегда пишет парсеры с нуля. В современной экосистеме Python существуют готовые решения для Data-Driven Testing.

#### 1. Использование библиотеки `parametrize_from_file`
Эта библиотека стала стандартом де-факто для выноса тестовых данных в YAML/JSON файлы [2]. Она оборачивает сложную логику `pytest_generate_tests` в элегантный декоратор.

```python
import parametrize_from_file

# Библиотека сама найдет файл test_payment.yaml, распасит его 
# и сгенерирует сотни тест-кейсов с индивидуальными маркерами (tags).
@parametrize_from_file
def test_payment_gateway(amount, currency, expected_status):
    response = api_client.process_payment(amount, currency)
    assert response.status == expected_status
```

#### 2. Интеграция Pydantic для валидации схем (Schema Validation)
Огромной проблемой динамических JSON/YAML файлов являются опечатки (кто-то написал `curency` вместо `currency`). Если передать такие данные в `metafunc.parametrize`, автотест упадет с непонятной ошибкой [4]. 
В 2026 году **строгим правилом** является валидация всех внешних тестовых данных через `Pydantic` *до* того, как они попадут в пайплайн кодогенерации [4]. Если YAML файл сформирован неверно, Pytest мгновенно остановит прогон с четким указанием строки ошибки в файле.

---

### Часть 3. Best Practices & Anti-patterns (SDET Контекст)

#### ❌ Anti-patterns (Архитектурные ошибки)
1.  **Чтение БД или тяжелые API-запросы внутри `pytest_generate_tests`:** 
    Хук кодогенерации работает на этапе сбора (Collection). Если вы попытаетесь выгрузить данные для параметризации долгим SQL-запросом, вы заблокируете весь пайплайн. Фаза "Collect" должна занимать миллисекунды [1].
2.  **Отсутствие `ids` при параметризации:** 
    Если вы передаете в `metafunc.parametrize` сырые JSON-объекты без параметра `ids`, то в отчете Allure или консоли вы увидите нечитаемые имена тестов вроде `test_flow[<dict object at 0x123>]` [1][4]. Всегда генерируйте понятные ID (например, `ids=[f"user_{u['id']}" for u in data]`).
3.  **Использование CSV для сложных структур:** 
    В 2026 году использование плоских CSV-файлов для параметризации микросервисов считается моветоном. Сложные полезные нагрузки (Payloads) требуют вложенности, которую могут обеспечить только JSON или YAML [3].

#### ✅ Best Practices (Стандарты 2026)
1.  **Indirect Parametrization (Косвенная параметризация):** 
    Если данные из JSON нужно сначала как-то "приготовить" (например, достать по ID пользователя из базы реальный токен), используйте `indirect=True` [4]. В этом случае Pytest сначала передаст данные из JSON в вашу Фикстуру, фикстура "превратит" их в реальный объект, и только потом отдаст в Тест [1][4].
2.  **Двойная параметризация (Матрица тестов):** 
    Динамическую параметризацию можно стакать (Stacking). Вы можете динамически выгрузить список из 10 White Label брендов из JSON, а затем скомбинировать их с 3 типами браузеров. Pytest автоматически сгенерирует $10 \times 3 = 30$ тест-кейсов [3].
3.  **Тестирование самих тестовых данных:** 
    В крупном проекте Data-файлы (YAML/JSON) хранятся в отдельном репозитории. Senior SDET настраивает CI/CD так, что при любом коммите в репозиторий с данными запускается отдельный линтер (например, `yamllint`), который проверяет валидность структуры, прежде чем позволить этим данным попасть в автотесты.

***

### 📚 Sources (Источники)
1. [How to parametrize tests with json array test data using pytest in python? (Stack Overflow)](https://stackoverflow.com/questions/57697473/how-to-parametrize-tests-with-json-array-test-data-using-pytest-in-python)
2. [Parametrizing multiple tests dynamically in Python (Stack Overflow)](https://stackoverflow.com/questions/66896898/parametrizing-multiple-tests-dynamically-in-python)
3. [Dynamically Generating Tests with Pytest Parametrization (Packet Coders, 2024)](https://www.packetcoders.io/dynamically-generating-tests-with-pytest-parametrization/)
4. [How to Use pytest Parametrize (OneUptime, Feb 2026 Update)](https://oneuptime.com/blog/pytest-parametrize)
5. [Advanced Pytest Patterns: Harnessing the Power of Parametrization and Factory Methods (Fiddler.ai, 2024)](https://www.fiddler.ai/blog/advanced-pytest-patterns-parametrization-and-factory-methods)
