# Что такое Итератор? / What is an Iterator?

**RU:**
**Итератор (Iterator)** — это объект, который представляет собой поток данных. Он помнит, на какой позиции находится, и умеет выдавать следующий элемент по запросу.

В отличие от списка, итератор **не хранит** все элементы в памяти сразу. Он вычисляет (или достает) их "лениво" — по одному за раз. Это делает итераторы идеальными для работы с бесконечными последовательностями или огромными файлами.

**EN:**
An **Iterator** is an object representing a stream of data. It remembers its current position and can yield the next item upon request.

Unlike a list, an iterator **does not store** all items in memory at once. It computes (or retrieves) them "lazily" — one by one. This makes iterators ideal for handling infinite sequences or massive files.

---

### 1. Протокол итератора / The Iterator Protocol

Чтобы объект считался итератором, он должен реализовывать два магических метода:

1.  **`__iter__`**: Возвращает сам объект (`return self`). Это нужно, чтобы итератор можно было использовать в циклах `for`.
2.  **`__next__`**: Возвращает следующий элемент. Если элементов больше нет, **обязан** вызвать исключение `StopIteration`.

#### Пример реализации "Хардкорный" / "Hardcore" Implementation Example

Это классический способ создания итератора через класс. Именно так они работают "под капотом".

```python
from collections.abc import Iterator  # Best Practice для Python 3.9+

class Counter(Iterator[int]):
    """
    Итератор, который считает от low до high.
    Iterator that counts from low to high.
    """
    def __init__(self, low: int, high: int):
        self.current = low
        self.high = high

    def __iter__(self):
        # Итератор должен возвращать сам себя
        # The iterator must return itself
        return self

    def __next__(self) -> int:
        # Если дошли до конца — останавливаемся
        # If we reached the end — stop
        if self.current > self.high:
            raise StopIteration
        
        # Сохраняем текущее значение, обновляем счетчик и возвращаем сохраненное
        # Save current value, update counter, and return the saved value
        result = self.current
        self.current += 1
        return result

# Использование / Usage:
counter = Counter(1, 3)

print(next(counter))  # 1
print(next(counter))  # 2
print(next(counter))  # 3
# print(next(counter))  # StopIteration Exception!
```

---

### 2. Iterable vs Iterator (Важное различие!)

Это самый частый вопрос на собеседованиях.

*   **Iterable (Итерируемый объект):** Это "коробка с данными" (список, строка, файл). Он **не знает**, где мы сейчас находимся. У него есть метод `__iter__()`, который создает и возвращает *новый* итератор.
*   **Iterator (Итератор):** Это "палец", который указывает на конкретный элемент в коробке. У него есть метод `__next__()`, чтобы сдвигаться дальше.

**Аналогия:**
*   **Iterable** — это Книга.
*   **Iterator** — это Закладка в книге.

Вы можете вставить 5 разных закладок (итераторов) в одну книгу (iterable), и каждая будет на своей странице.

```python
# List is Iterable, but NOT an Iterator
numbers = [1, 2, 3]

# iter(numbers) создает итератор
it1 = iter(numbers)
it2 = iter(numbers)

print(next(it1)) # 1
print(next(it1)) # 2
print(next(it2)) # 1 (Второй итератор независим! / Second iterator is independent!)
```

---

### 3. Современные типы и Best Practices (2024/2025)

#### Deprecation of `typing.Iterator`
В старых гайдах вы увидите импорт `from typing import Iterator`.
Начиная с **Python 3.9+**, это считается устаревшим (deprecated).
Официальная рекомендация на 2025 год — использовать **`collections.abc`**.

| Old Way (Python < 3.9) | Modern Way (Python 3.9+) |
| :--- | :--- |
| `from typing import Iterable` | `from collections.abc import Iterable` |
| `from typing import Iterator` | `from collections.abc import Iterator` |

#### Зачем нам `collections.abc`?
Стандарт `collections.abc` (Abstract Base Classes) позволяет проверять типы более надежно.

```python
from collections.abc import Iterator

# Проверка, является ли объект итератором
# Checking if an object is an iterator
data = iter([1, 2, 3])
if isinstance(data, Iterator):
    print("This is an iterator!")
```

---

### 4. Опасности итераторов / Pitfalls

**RU:**
Итераторы — это "одноразовые" объекты. Прочитав итератор до конца, вы не можете "перемотать" его назад. Если вам нужно пройтись по данным снова, нужно создать новый итератор.

**EN:**
Iterators are "one-off" objects. Once consumed, they cannot be "rewound" or reset. If you need to iterate over the data again, you must create a new iterator.

```python
squares = (x*x for x in range(3)) # Генератор (разновидность итератора)

print(list(squares)) # [0, 1, 4]
print(list(squares)) # [] <- ПУСТО! Итератор исчерпан / EMPTY! Iterator exhausted
```
