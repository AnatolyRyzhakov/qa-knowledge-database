## Что такое Безопасная итерация? / What is Safe Iteration?

**RU:**
**Безопасная итерация** — это подход к работе с коллекциями, который позволяет изменять их структуру (добавлять или удалять элементы) без возникновения ошибок (`RuntimeError`) и без нарушения логики обхода (пропуска элементов).

Основное правило: **Нельзя менять размер коллекции, пока вы по ней итерируетесь.**

**EN:**
**Safe Iteration** is an approach to handling collections that allows you to modify their structure (add or remove items) without causing errors (`RuntimeError`) or logic bugs (skipped items).

The golden rule: **Do not change the size of a collection while iterating over it.**

---

### 1. Под капотом: Почему это ломается? / Under the Hood: Why does it break?

Чтобы понять проблему, нужно взглянуть на то, как работает цикл `for` в Python.

#### 1.1. Проблема списков: Сдвиг индексов (Index Shifting)

**RU:**
Списки в Python — это массивы ссылок. Когда вы запускаете `for item in my_list`, Python создает **итератор**, который хранит внутренний счетчик (индекс), указывающий на текущий элемент.

Представим список: `[10, 20, 30, 40]`. Мы хотим удалить всё, что `>= 20`.

1.  **Итерация 1:** Индекс итератора = `0`. Элемент `10`. Оставляем.
2.  **Итерация 2:** Индекс итератора = `1`. Элемент `20`. **Удаляем!**
    *   Что происходит в памяти? Элементы сдвигаются влево, чтобы закрыть дыру.
    *   Список теперь выглядит так: `[10, 30, 40]`.
    *   Элемент `30` теперь находится по индексу `1`.
3.  **Итерация 3:** Индекс итератора увеличивается и становится `2`.
    *   Python берет элемент по индексу `2` из *нового* состояния списка.
    *   Это элемент `40`.
    *   **Результат:** Элемент `30` (который встал на место удаленного) был **пропущен**.

**EN:**
Lists in Python are arrays of references. When you run `for item in my_list`, Python creates an **iterator** that keeps an internal counter (index) pointing to the current item.

Imagine a list: `[10, 20, 30, 40]`. We want to remove everything `>= 20`.

1.  **Iteration 1:** Iterator index = `0`. Item `10`. Keep it.
2.  **Iteration 2:** Iterator index = `1`. Item `20`. **Delete it!**
    *   What happens in memory? Items shift to the left to fill the gap.
    *   List is now: `[10, 30, 40]`.
    *   Item `30` is now at index `1`.
3.  **Iteration 3:** Iterator index increments to `2`.
    *   Python grabs the item at index `2` from the *new* list state.
    *   This is item `40`.
    *   **Result:** Item `30` (which moved into the deleted spot) was **skipped**.

#### 1.2. Проблема словарей: Изменение структуры (Hash Table Resizing)

**RU:**
Словари работают на основе хэш-таблиц. Порядок ключей и их расположение в памяти зависят от размера таблицы. Если вы удаляете ключ, внутренняя структура таблицы может измениться. Итератор теряется, и Python выбрасывает `RuntimeError`.

**EN:**
Dictionaries work on hash tables. The order of keys and their memory location depend on the table size. If you delete a key, the internal table structure might change. The iterator gets lost, and Python raises a `RuntimeError`.

---

### 2. Решение через копирование: `.copy()`, `[:]`, `list()`

Самый надежный способ — бежать по **копии** коллекции, а менять **оригинал**. В Python поверхностная копия (Shallow Copy) создает новый контейнер, но элементы внутри остаются ссылками на те же объекты.

#### 2.1. Использование среза `[:]` (Slicing)

**RU:**
Синтаксис `[:]` создает полный срез списка, что технически является созданием нового списка с теми же элементами. Это очень быстрый и популярный способ в старом Python коде.

**EN:**
The `[:]` syntax creates a full slice of the list, which technically creates a new list with the same elements. It is a very fast and popular method in older Python code.

```python
users = ["Alice", "Bob", "Charlie", "Dave"]

# users[:] создает новый список в памяти
for user in users[:]:
    if len(user) < 4:
        # Мы удаляем из users, но цикл идет по анонимной копии
        users.remove(user)

print(users) # ['Alice', 'Charlie', 'Dave']
```

#### 2.2. Использование конструктора `list()`

**RU:**
Вызов `list(iterable)` создает новый список на основе переданного итерируемого объекта. Это работает не только со списками, но и со словарями (создает список ключей), множествами и генераторами.

**EN:**
Calling `list(iterable)` creates a new list based on the provided iterable object. This works not only with lists but also with dictionaries (creates a list of keys), sets, and generators.

```python
data = {"a": 1, "b": 2, "c": 3}

# list(data) создает ['a', 'b', 'c']
# Если написать просто `for key in data:`, будет ошибка RuntimeError
for key in list(data):
    if data[key] < 2:
        del data[key]

print(data) # {'b': 2, 'c': 3}
```

#### 2.3. Метод `.copy()`

**RU:**
Начиная с Python 3.3, у списков появился явный метод `.copy()`. Он делает то же самое, что и `[:]`, но код становится более читаемым для новичков (сразу понятно, что происходит копирование).

**EN:**
Since Python 3.3, lists have an explicit `.copy()` method. It does the same thing as `[:]`, but the code becomes more readable for beginners (it's immediately obvious that copying is happening).

```python
items = [1, 2, 3, 4]

for item in items.copy():
    if item % 2 == 0:
        items.remove(item)
```

---

### 3. Глубокая копия (Deep Copy)

**RU:**
`copy.deepcopy()` создает полную рекурсивную копию всего дерева объектов. Если список содержит другие списки, они тоже будут скопированы.
**Для безопасной итерации это обычно избыточно.** Это тратит много памяти и времени. Используйте `deepcopy` только если вы планируете менять *внутренности* вложенных объектов так, чтобы это не затронуло оригинал.

**EN:**
`copy.deepcopy()` creates a full recursive copy of the entire object tree. If a list contains other lists, they are copied too.
**For safe iteration, this is usually overkill.** It wastes memory and time. Use `deepcopy` only if you plan to modify the *internals* of nested objects without affecting the original.

```python
import copy

# Список списков
matrix = [[1, 2], [3, 4]]

# Здесь deepcopy не нужен, достаточно matrix[:]
# Мы удаляем сам подсписок, а не меняем числа внутри него
for row in copy.deepcopy(matrix):
    if sum(row) < 5:
        matrix.remove(row)
```

---

### 4. Основные стратегии без копирования / Strategies without Copying

#### 4.1. Идиоматичный Python (List Comprehension)

**RU:**
Вместо удаления элементов из старого списка, создайте новый. Это быстрее и чище.

**EN:**
Instead of removing items from the old list, create a new one. It is faster and cleaner.

```python
numbers = [1, 2, 3, 4, 5, 6]
# Фильтрация: оставляем только нечетные
numbers = [n for n in numbers if n % 2 != 0]
```

#### 4.2. Итерация с конца (Reversed Iteration)

**RU:**
При удалении элемента сдвигается только "хвост" списка. Если идти с конца, индексы еще не проверенных элементов (в начале списка) не меняются. Это экономит память, так как не создается копия списка.

**EN:**
When deleting an item, only the "tail" of the list shifts. If iterating backwards, the indices of the unchecked items (at the start) do not change. This saves memory as no list copy is created.

```python
data = [10, 20, 30, 40]

# range(start, stop, step) -> идем от последнего индекса к 0
for i in range(len(data) - 1, -1, -1):
    if data[i] >= 20:
        del data[i]

print(data) # [10]
```

---

### Резюме / Summary

| Метод / Method | Безопасность / Safety | Скорость / Speed | Примечание / Note |
| :--- | :--- | :--- | :--- |
| `for x in list:` + `remove()` | **Опасно** | - | Приводит к пропуску элементов или ошибкам. |
| `list[:]`, `.copy()`, `list()` | **Безопасно** | Средняя | Стандартный подход. Создает копию в памяти. |
| `[x for x in list]` | **Безопасно** | Высокая | Самый быстрый и "питоничный" способ. |
| `reversed()` / `range()` | **Безопасно** | Высокая | Экономит память (не создает копию). Подходит только для списков по индексу. |
