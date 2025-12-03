# Что такое Генератор? / What is a Generator?

**RU:**
**Генератор (Generator)** — это простой способ создания итераторов.
Если итератор — это "ручная сборка" (через класс с методами `__iter__` и `__next__`), то генератор — это "автоматическая фабрика".

Вы просто пишете обычную функцию, но вместо `return` используете ключевое слово **`yield`**. Python автоматически превращает эту функцию в итератор, который умеет "замораживать" свое состояние.

**EN:**
A **Generator** is a simple way to create iterators.
If an iterator is "manual assembly" (using a class with `__iter__` and `__next__`), then a generator is an "automated factory".

You write a standard function, but instead of `return`, you use the keyword **`yield`**. Python automatically converts this function into an iterator that can "freeze" its state.

---

### 1. `yield` vs `return`

Это главное отличие, которое нужно понять.

*   **`return`**: Вычисляет значение, **уничтожает** локальные переменные и выходит из функции навсегда.
*   **`yield`**: "Отдает" значение, **замораживает** состояние функции (запоминает переменные) и ждет, пока её позовут снова (через `next()`).

#### Пример / Example

```python
from collections.abc import Generator

def simple_generator() -> Generator[int, None, None]:
    print("Start")
    yield 1
    print("Resumed")
    yield 2
    print("Finish")

# Функция не выполняется сразу! Она возвращает объект-генератор.
# The function doesn't run immediately! It returns a generator object.
gen = simple_generator()

print(next(gen)) 
# Output: 
# Start
# 1

print(next(gen)) 
# Output: 
# Resumed
# 2
```

---

### 2. Generator Expressions (Генераторные выражения)

**RU:**
Это синтаксический сахар. Похоже на List Comprehension, но используются **круглые скобки `()`**.
Они не создают список в памяти, а возвращают генератор, который вычисляет значения на лету.

**EN:**
This is syntactic sugar. Similar to List Comprehension, but uses **parentheses `()`**.
They do not create a list in memory but return a generator that computes values on the fly.

| Тип / Type | Синтаксис / Syntax | Память / Memory |
| :--- | :--- | :--- |
| **List Comp** | `[x*2 for x in data]` | **Высокое**. Весь список хранится в RAM. |
| **Gen Exp** | `(x*2 for x in data)` | **Минимальное**. Хранится только формула. |

```python
import sys

# List Comprehension (Миллион чисел сразу в памяти)
my_list = [i for i in range(1000000)]
print(sys.getsizeof(my_list))  # ~8 MB

# Generator Expression (Ленивое вычисление)
my_gen = (i for i in range(1000000))
print(sys.getsizeof(my_gen))   # ~112 Bytes (Const!)
```

---

### 3. Продвинутая техника: `yield from`

**RU:**
Конструкция `yield from` появилась для упрощения вложенных генераторов. Она позволяет "делегировать" выполнение другому итератору. Это работает как "выпрямление" (flattening) вложенных структур.

**EN:**
The `yield from` syntax was introduced to simplify nested generators. It allows "delegating" execution to another iterator. It acts like "flattening" nested structures.

**До / Before:**
```python
def sub_generator():
    yield "A"
    yield "B"

def main_generator():
    # Нам нужно вручную перебирать вложенный генератор
    for item in sub_generator():
        yield item
    yield "End"
```

**После / After:**
```python
def main_generator():
    # Python сам "высасывает" всё из sub_generator и отдает наружу
    yield from sub_generator()
    yield "End"

print(list(main_generator())) # ['A', 'B', 'End']
```

---

### 4. Типизация (Modern Typing 2024+)

В современном Python (3.9+) мы используем `collections.abc`.

Стандартная аннотация выглядит так:
`Generator[YieldType, SendType, ReturnType]`

*   **YieldType**: Что генератор отдает через `yield` (обычно это самое важное).
*   **SendType**: Что мы можем отправить внутрь через `.send()` (часто `None`).
*   **ReturnType**: Что генератор возвращает через `return` в конце (часто `None`).

**Упрощение (Best Practice):**
Если вы просто читаете данные из генератора (как в 99% случаев), можно использовать **`Iterator[Type]`**. Это проще читать.

```python
from collections.abc import Generator, Iterator

# Строгая аннотация (Strict)
def strict_gen() -> Generator[int, None, None]:
    yield 1

# Упрощенная аннотация (Simple / Reader-only)
def simple_gen() -> Iterator[int]:
    yield 1
```

---

### 5. Зачем это в AutoQA? / Why in AutoQA?

1.  **Data Driven Testing:** Генераторы идеальны для создания миллионов тестовых данных, не забивая память.
2.  **Парсинг логов:** Если нужно найти ошибку в лог-файле размером 10 ГБ, генератор позволит читать его построчно, потребляя всего пару килобайт памяти.
3.  **Ожидание событий (Polling):** Генераторы часто используют для создания умных "ожидалок" (waiters), которые проверяют условие раз в секунду.

### 6. `yield` в Pytest (Setup & Teardown)

**RU:**
В автоматизации тестирования `yield` чаще всего встречается в **фикстурах**. Здесь он работает как разделитель: всё, что написано **до** `yield`, выполняется перед тестом (Setup), а всё, что **после** — после теста (Teardown).

Это позволяет гарантированно удалять тестовые данные или закрывать браузер, даже если сам тест упал с ошибкой.

**EN:**
In test automation, `yield` is most commonly found in **fixtures**. Here acts as a separator: everything written **before** `yield` runs before the test (Setup), and everything **after** runs after the test (Teardown).

This ensures that test data is deleted or the browser is closed, even if the test itself fails.

#### Пример 1: Создание и удаление пользователя (API) / Create & Delete User

Самый частый сценарий: создаем временного юзера для теста, а потом удаляем его, чтобы не мусорить в базе.

```python
import pytest

@pytest.fixture
def temporary_user(api_client):
    # --- SETUP (Перед тестом) ---
    print("Создаем пользователя...")
    user_id = api_client.create_user(name="TestBot")
    
    # Отдаем ID в тест и "замораживаемся"
    yield user_id 
    
    # --- TEARDOWN (После теста) ---
    # Этот код выполнится в любом случае
    print(f"Удаляем пользователя {user_id}...")
    api_client.delete_user(user_id)

# В тесте:
def test_profile_update(temporary_user):
    # temporary_user равен user_id
    assert api_client.get_user(temporary_user).name == "TestBot"
```

#### Пример 2: Управление браузером (UI) / Browser Management

Здесь мы открываем браузер, отдаем его тесту, а потом закрываем.

```python
from selenium import webdriver
import pytest

@pytest.fixture
def driver():
    # --- SETUP ---
    print("Запускаем Chrome...")
    driver_instance = webdriver.Chrome()
    driver_instance.get("https://google.com")
    
    # Передаем объект драйвера в тест
    yield driver_instance
    
    # --- TEARDOWN ---
    print("Закрываем браузер...")
    driver_instance.quit()

def test_search(driver):
    # Пользуемся драйвером
    assert "Google" in driver.title
```

#### Пример 3: Изменение настроек (Config Toggle)

Иногда нужно временно включить какую-то настройку, а потом вернуть как было. Здесь `yield` ничего не возвращает, просто ставит паузу.

```python
@pytest.fixture
def maintenance_mode(admin_api):
    # --- SETUP ---
    # Запоминаем старое состояние
    was_enabled = admin_api.is_maintenance_on()
    # Включаем режим обслуживания
    admin_api.set_maintenance(True)
    
    yield # Просто передаем управление тесту (ничего не возвращаем)
    
    # --- TEARDOWN ---
    # Возвращаем всё как было
    if not was_enabled:
        admin_api.set_maintenance(False)
```
