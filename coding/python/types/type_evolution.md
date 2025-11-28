# Эволюция типов/Types Evolution

## 1. `List` vs `list` / `Dict` vs `dict`

**RU:**
До Python 3.9 встроенные типы (`list`, `dict`, `tuple`) не поддерживали указание содержимого через квадратные скобки (generics). Если вы писали `list[int]`, Python выдавал ошибку.
Поэтому нам приходилось импортировать специальные обертки из модуля `typing`: `List`, `Dict`, `Tuple`.

**Начиная с Python 3.9 (PEP 585):** Встроенные типы научились принимать аргументы. Типы из `typing` теперь считаются **устаревшими (deprecated)**.

**EN:**
Before Python 3.9, built-in types (`list`, `dict`, `tuple`) did not support generics (specifying content via square brackets). Writing `list[int]` caused an error.
That's why we had to import wrappers from `typing`: `List`, `Dict`, `Tuple`.

**Since Python 3.9 (PEP 585):** Built-in types support generics natively. The `typing` counterparts are now **deprecated**.

### Сравнение / Comparison

**Устарело (Old way, Python < 3.9):**
```python
from typing import List, Dict

def process_items(items: List[int]) -> Dict[str, int]:
    return {"count": len(items)}
```

**Современно (Modern way, Python 3.9+):**
```python
# Импорты больше не нужны! / No imports needed!
def process_items(items: list[int]) -> dict[str, int]:
    return {"count": len(items)}
```

**Рекомендация:** Всегда используйте `list`, `dict`, `tuple`, `set`, `frozenset` с маленькой буквы.

---

## 2. `Optional[Type]` vs `Type | None`

**RU:**
`Optional` — это просто красивое название для объединения типа с `None`.
`Optional[int]` на самом деле означает `Union[int, None]`.

**Начиная с Python 3.10 (PEP 604):** Появился новый оператор объединения `|`. Теперь `Optional` и `Union` можно заменить на более короткую запись.

**EN:**
`Optional` is just an alias for combining a type with `None`.
`Optional[int]` actually means `Union[int, None]`.

**Since Python 3.10 (PEP 604):** The new union operator `|` was introduced. Now `Optional` and `Union` can be replaced with a shorter syntax.

### Сравнение / Comparison

**Устарело (Old way):**
```python
from typing import Optional, Union

# Может быть числом или None
def find_user(user_id: int) -> Optional[str]:
    ...

# Может быть строкой или числом
def parse(value: Union[int, str]):
    ...
```

**Современно (Modern way, Python 3.10+):**
```python
# Optional[str] -> str | None
def find_user(user_id: int) -> str | None:
    ...

# Union[int, str] -> int | str
def parse(value: int | str):
    ...
```

**Важное замечание про `None`:**
Аннотация `x: None` используется только тогда, когда переменная *всегда* равна `None` (или функция ничего не возвращает). Если переменная *может* быть `None`, а может чем-то еще — используйте `| None`.

---

## 3. Другие похожие случаи / Other Similar Cases

В Python есть еще один слой типов — **Абстрактные коллекции** (Abstract Base Classes). Они нужны, когда вам не важно, передали вам именно `list` или `tuple`, главное — чтобы по объекту можно было пройтись циклом.

### `Iterable`, `Sequence`, `Callable`

**RU:**
Раньше их тоже брали из `typing`. Теперь (с 3.9+) их нужно брать из `collections.abc`.
Это важно, потому что `typing.Iterable` скоро удалят.

**EN:**
Previously, these were also imported from `typing`. Now (since 3.9+), they should be imported from `collections.abc`.
This is important because `typing.Iterable` will be removed eventually.

| Тип / Type | Старый импорт (Old) | Новый импорт (New, 3.9+) | Описание / Description |
| :--- | :--- | :--- | :--- |
| **Итерируемый** | `typing.Iterable` | `collections.abc.Iterable` | Можно использовать в `for` (список, кортеж, генератор). |
| **Последовательность** | `typing.Sequence` | `collections.abc.Sequence` | Можно использовать `for` + есть индексы `[0]` и `len()`. |
| **Отображение** | `typing.Mapping` | `collections.abc.Mapping` | Ведет себя как словарь (ключ-значение). |
| **Вызываемый** | `typing.Callable` | `collections.abc.Callable` | Можно вызвать как функцию `()`. |

### Пример использования / Usage Example

```python
from collections.abc import Iterable, Callable

# Принимаем любой итерируемый объект (list, tuple, set, generator)
def process_data(data: Iterable[int]):
    for item in data:
        print(item)

# Принимаем функцию (callback)
def execute(action: Callable[[], None]):
    action()
```

---

## Итоговая таблица / Summary Table

Эта таблица поможет быстро перевести старый код на новый лад.

| Что хотим описать? | Было (Python < 3.9) | Стало (Python 3.9/3.10+) |
| :--- | :--- | :--- |
| Список чисел | `List[int]` | `list[int]` |
| Словарь | `Dict[str, int]` | `dict[str, int]` |
| Кортеж | `Tuple[int, int]` | `tuple[int, int]` |
| Множество | `Set[str]` | `set[str]` |
| Или то, или то | `Union[int, str]` | `int \| str` (3.10+) |
| Значение или None | `Optional[int]` | `int \| None` (3.10+) |
| Любой итератор | `typing.Iterable` | `collections.abc.Iterable` |

### Совместимость (Backward Compatibility)

**RU:**
Если вы используете Python 3.8 или 3.9, но хотите использовать красивый синтаксис `|` (пайп) вместо `Union`, добавьте этот магический импорт в самое начало файла:

**EN:**
If you are using Python 3.8 or 3.9 but want to use the clean `|` (pipe) syntax instead of `Union`, add this magic import at the very top of your file:

```python
from __future__ import annotations

def func(x: int | str) -> list[str]:  # Работает даже на 3.8 / Works even on 3.8
    ...
```
