## Что такое Tuple? / What is a Tuple?

**RU:**
**Tuple (Кортеж)** — это неизменяемая (immutable) упорядоченная последовательность элементов.
Визуально он отличается от списка круглыми скобками `()`, но главное отличие — в его поведении и предназначении.

Если список (`list`) — это "папка с файлами", куда можно докладывать новые документы, то кортеж (`tuple`) — это "запечатанный конверт" или "строка в базе данных". После создания вы не можете добавить, удалить или заменить элементы внутри него.

**EN:**
A **Tuple** is an immutable, ordered sequence of elements.
Visually, it is distinguished from a list by parentheses `()`, but the main difference lies in its behavior and purpose.

If a `list` is a "folder" where you can add files, a `tuple` is a "sealed envelope" or a "database row". Once created, you cannot add, remove, or replace elements inside it.

---

### 1. Сравнение: Tuple vs List vs Set

| Характеристика | Tuple `()` | List `[]` | Set `{}` |
| :--- | :--- | :--- | :--- |
| **Изменяемость (Mutable)** | Нет (Immutable) | Да | Да |
| **Порядок (Ordered)** | Да | Да | Нет |
| **Дубликаты** | Да | Да | Нет (Только уникальные) |
| **Скорость (Performance)** | Быстрее | Медленнее | Быстро (поиск) |
| **Память (Memory)** | Меньше | Больше | Больше |
| **Смысл (Semantics)** | Структура (Запись) | Коллекция (Массив) | Уникальный набор |

---

### 2. Под капотом: Почему Tuple быстрее? / Under the Hood: Why is Tuple faster?

Это ключевой вопрос для Senior-уровня. Разница не только в запрете на изменение.

#### 1. Выделение памяти (Memory Allocation)
*   **List:** Это динамический массив. Чтобы вы могли делать `.append()`, Python выделяет память "с запасом" (Over-allocation). Сначала под 4 элемента, потом под 8, 16 и т.д. Плюс список хранит отдельно сам объект списка и отдельно ссылку на массив указателей.
*   **Tuple:** Имеет фиксированный размер. Python точно знает, сколько памяти нужно, и выделяет её **одним цельным блоком**.

#### 2. Кэширование (Resource Caching)
Python имеет внутренний механизм оптимизации для малых кортежей (до 20 элементов). Когда кортеж больше не нужен, Python не уничтожает его память сразу, а откладывает в "free list".
Если вам понадобится создать новый кортеж такого же размера, Python не будет просить память у ОС (что долго), а мгновенно возьмет готовый блок из "кармана". У списков такой агрессивной оптимизации нет.

**Доказательство:**
```python
import sys

t = (1, 2, 3, 4, 5)
l = [1, 2, 3, 4, 5]

print(sys.getsizeof(t))  # ~80 bytes (меньше)
print(sys.getsizeof(l))  # ~120 bytes (больше из-за запаса)
```

---

### 3. Семантическая разница: Структура vs Коллекция

Это то, что отличает профессионала от новичка.

**RU:**
*   **List (Гомогенный):** Обычно используется для хранения данных **одного типа**.
    *   *Пример:* Список пользователей `[User1, User2, User3]`. Длина списка может меняться, но суть элементов одна.
*   **Tuple (Гетерогенный):** Обычно используется как **Структура** (Запись), где позиция элемента имеет значение.
    *   *Пример:* Координаты `(x, y)`, запись в БД `(ID, Name, Date)`. Здесь `element[0]` — это всегда ID, а `element[1]` — всегда Имя. Удалить элемент отсюда нельзя, иначе сломается смысл.

**EN:**
*   **List (Homogeneous):** Usually used to store data of the **same type**.
    *   *Example:* A list of users `[User1, User2, User3]`. The length changes, but the entity type is the same.
*   **Tuple (Heterogeneous):** Usually used as a **Structure** (Record), where the position involves meaning.
    *   *Example:* Coordinates `(x, y)`, DB row `(ID, Name, Date)`. Here `element[0]` is always ID. Removing an item breaks the logic.

---

### 4. Практические примеры / Practical Use Cases

#### Кейс 1: Ключи словаря (Dictionary Keys)
Списки нельзя использовать как ключи, потому что они изменяемы (не hashable). Кортежи — можно.

```python
# Нужно хранить локацию (широта, долгота) -> погода
weather_cache = {}

location = (55.75, 37.61) # Tuple
weather_cache[location] = "Sunny" # Works!

# location_list = [55.75, 37.61]
# weather_cache[location_list] = "Rainy" # TypeError: unhashable type: 'list'
```

#### Кейс 2: Возврат нескольких значений (Multiple Return Values)
Когда функция возвращает `return a, b`, она на самом деле возвращает кортеж `(a, b)`.

```python
def get_screen_resolution():
    return 1920, 1080  # Implicit tuple creation

width, height = get_screen_resolution() # Unpacking
```

#### Кейс 3: Защита от дурака (Read-only Configuration)
Если у вас есть список серверов или портов, которые **никогда** не должны меняться во время работы программы, используйте кортеж. Это гарантирует, что никакой случайный код (или Junior разработчик) не сделает `.append()` или `.pop()`.

```python
ALLOWED_PORTS = (80, 443, 8080)
# ALLOWED_PORTS.append(22) # AttributeError
```

---

### 5. NamedTuple — Современная замена (Modern Replacement)

В современном Python (особенно в AutoQA для DTO) обычные кортежи `(1, "Alice")` часто заменяют на `NamedTuple` или `Dataclass` (frozen). Это делает код читаемым.

```python
from typing import NamedTuple

class UserDTO(NamedTuple):
    id: int
    name: str

# Это всё ещё кортеж! Он быстрый и ест мало памяти.
user = UserDTO(1, "Alice")

print(user[0])   # 1 (Как обычный кортеж)
print(user.name) # "Alice" (Удобно!)
```
