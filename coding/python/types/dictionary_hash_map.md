## Что такое Hash Map? / What is a Hash Map?

**RU:**
Hash Map (в Python это `dict`) — это структура данных, которая хранит пары "ключ-значение".
Её главная суперсила — **скорость**. Поиск значения по ключу происходит за **O(1)** (мгновенно), независимо от того, сколько элементов в словаре: 10 или 10 миллионов.

Это работает благодаря **хэшированию**: Python не перебирает ключи один за другим (как в списке), а мгновенно вычисляет адрес ячейки в памяти, где лежит значение, используя математическую формулу.

**EN:**
A Hash Map (in Python, `dict`) is a data structure that stores "key-value" pairs.
Its main superpower is **speed**. Looking up a value by key takes **O(1)** (instantly), regardless of whether the dictionary has 10 items or 10 million.

This works thanks to **hashing**: Python doesn't iterate over keys one by one (like in a list) but instantly calculates the memory address where the value is stored using a mathematical formula.

---

### 1. Как это работает "под капотом"? / Under the Hood

До Python 3.6 словари были "разреженными" массивами (занимали много памяти).
**Современный Python (3.7+) использует "Compact Dict" (Компактный словарь).**

Эта реализация состоит из двух массивов:
1.  **Indices (Индексы):** Маленькая "разреженная" таблица, где хранятся только ссылки (индексы) на второй массив.
2.  **Entries (Записи):** Плотный массив, куда данные записываются подряд (в порядке добавления).

#### Визуализация / Visualization

Допустим, мы делаем:
```python
data = {"apple": 1, "banana": 2}
```

**Шаг 1. Вычисление хэша:**
*   `hash("apple")` -> `12345`
*   `hash("banana")` -> `67890`

**Шаг 2. Определение индекса (Indices Array):**
Берем остаток от деления хэша на размер таблицы (например, 8).
*   `12345 % 8` = `1`
*   `67890 % 8` = `2`

**Шаг 3. Запись в память (Compact Storage):**

Массив **Entries** (хранит сами данные, заполняется подряд):
| Index (в Entries) | Hash | Key | Value |
| :--- | :--- | :--- | :--- |
| **0** | 12345 | "apple" | 1 |
| **1** | 67890 | "banana" | 2 |

Массив **Indices** (хэш-таблица, хранит ссылки на Entries):
| Index (в Hash Table) | Ссылка на Entries |
| :--- | :--- |
| 0 | None |
| 1 | **0** (ссылка на "apple") |
| 2 | **1** (ссылка на "banana") |
| ... | ... |

**Почему это круто?**
1.  **Память:** Мы не храним пустые тяжелые объекты в основном массиве.
2.  **Порядок:** Поскольку в `Entries` мы пишем подряд, современные словари **помнят порядок вставки** (Insertion Order). Раньше (до 3.7) порядок был случайным.

---

### 2. Коллизии (Collisions)

**RU:**
Коллизия — это когда у двух разных ключей получается **одинаковый индекс** в массиве `Indices`.
Пример: `hash("key1") % 8 == 3` и `hash("key2") % 8 == 3`.

В Python коллизии решаются методом **Открытой адресации (Open Addressing)**.

**Как это работает:**
Если ячейка `3` занята, Python не создает там список (как в Java), а начинает искать **следующую свободную ячейку** по хитрому алгоритму (Perturbation Shift). Он "прыгает" по таблице, пока не найдет пустое место.

Именно поэтому, если словарь заполнен на 2/3, Python автоматически увеличивает его размер и перестраивает все индексы, чтобы коллизий было меньше.

**EN:**
A collision occurs when two different keys result in the **same index** in the `Indices` array.
Example: `hash("key1") % 8 == 3` and `hash("key2") % 8 == 3`.

Python resolves collisions using **Open Addressing**.

**How it works:**
If cell `3` is occupied, Python doesn't create a linked list there (like Java does). Instead, it starts looking for the **next available cell** using a smart algorithm (Perturbation Shift). It "probes" the table until it finds an empty spot.

That is why, when a dictionary is 2/3 full, Python automatically resizes it and rebuilds all indices to reduce collisions.

---

### 3. Требования к ключам / Key Requirements

Ключом словаря может быть **только хэшируемый (hashable)** объект.
В Python это значит, что объект должен быть **неизменяемым (immutable)**.

| Тип | Можно использовать как ключ? | Почему? |
| :--- | :--- | :--- |
| `str`, `int`, `bool` | ДА | Они неизменяемы. |
| `tuple` | ДА | Если внутри кортежа тоже только неизменяемые типы. |
| `list` | НЕТ | Списки можно менять. Если список изменится, его хэш изменится, и словарь "потеряет" значение. |
| `dict` | НЕТ | Словари тоже изменяемы. |
| `set` | НЕТ | Изменяемы. (Используйте `frozenset`). |

**Пример ошибки:**
```python
my_dict = {}
my_list = [1, 2]

# TypeError: unhashable type: 'list'
my_dict[my_list] = "Error"
```

---

### 4. Сложность операций (Big O) / Complexity

| Операция | В среднем (Average) | В худшем случае (Worst Case) |
| :--- | :--- | :--- |
| **Поиск (`val = d[key]`)** | **O(1)** | O(n) |
| **Вставка (`d[key] = val`)** | **O(1)** | O(n) |
| **Удаление (`del d[key]`)** | **O(1)** | O(n) |

*   **Почему O(n) в худшем случае?**
    Это теоретическая ситуация, когда у ВСЕХ ключей случается коллизия (хэш-функция сломалась или подобрана злоумышленником). В реальности в Python такого почти не бывает благодаря качественной функции `hash()`.

---

### 5. Практические примеры / Practical Examples

#### 5.1. Значение по умолчанию (`.get`)

Не проверяйте наличие ключа через `if`. Используйте метод `.get()`.

**Плохо / Bad:**
```python
if "user_id" in data:
    user = data["user_id"]
else:
    user = None
```

**Хорошо / Good:**
```python
# Вернет None (или свой дефолт), если ключа нет
user = data.get("user_id", None)
```

#### 5.2. `setdefault` и `defaultdict`

Если вы собираете список значений внутри словаря.

```python
from collections import defaultdict

# Автоматически создает пустой список, если ключа нет
users_by_role = defaultdict(list)

users_by_role["admin"].append("Alice")
users_by_role["admin"].append("Bob")
# Не нужно писать: if "admin" not in dict: dict["admin"] = []
```

#### 5.3. Слияние словарей (Python 3.9+)

Новый удобный оператор `|`.

```python
default_settings = {"theme": "light", "notifications": True}
user_settings = {"theme": "dark"}

# user_settings перезапишет совпадения из default
config = default_settings | user_settings 
# Result: {'theme': 'dark', 'notifications': True}
```
