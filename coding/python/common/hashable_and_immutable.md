# Hashable и Immutable / Hashable vs Immutable

## 1. Неизменяемость (Immutability)

**RU:**
**Immutable (Неизменяемый)** объект — это объект, состояние которого **нельзя изменить** после его создания.
Если вы попытаетесь "изменить" такой объект (например, прибавить число или дописать строку), Python не изменит старый объект, а **создаст новый** в другой ячейке памяти.

**EN:**
An **Immutable** object is an object whose state **cannot be modified** after it is created.
If you try to "change" such an object (e.g., add a number or append to a string), Python will not modify the old object but will **create a new one** in a different memory location.

### Примеры типов / Type Examples

| Immutable (Нельзя менять) | Mutable (Можно менять) |
| :--- | :--- |
| `int`, `float`, `bool` | `list` (Список) |
| `str` (Строка) | `dict` (Словарь) |
| `tuple` (Кортеж) | `set` (Множество) |
| `frozenset` (Замороженное множество) | Пользовательские классы (обычно) |

#### Под капотом (Доказательство через `id`) / Under the hood

```python
# --- Immutable (int) ---
x = 10
print(id(x))  # Например: 4304922312

x = x + 1     # Мы не меняем 10 на 11. Мы создаем новый объект 11.
print(id(x))  # Другой адрес! Например: 4304922344

# --- Mutable (list) ---
lst = [1, 2]
print(id(lst)) # Например: 140234000

lst.append(3)  # Мы меняем тот же самый объект
print(id(lst)) # Тот же адрес: 140234000
```

---

## 2. Хэшируемость (Hashability)

**RU:**
**Hashable (Хэшируемый)** объект — это объект, который:
1.  Имеет хэш-значение (метод `__hash__`), которое **никогда не меняется** в течение жизни объекта.
2.  Может быть сравнен с другими объектами (метод `__eq__`).

**Золотое правило:** Если `a == b`, то обязательно `hash(a) == hash(b)`.

Именно хэшируемые объекты могут быть **ключами в словарях** (`dict`) и **элементами множеств** (`set`).

**EN:**
A **Hashable** object is an object that:
1.  Has a hash value (`__hash__` method) which **never changes** during its lifetime.
2.  Can be compared to other objects (`__eq__` method).

**The Golden Rule:** If `a == b`, then `hash(a) == hash(b)` must be true.

Only hashable objects can be used as **keys in dictionaries** (`dict`) and **elements in sets** (`set`).

---

### 3. Связь понятий / The Connection

Почему это важно? Вспомните, как работает Hash Map.
Чтобы найти значение, Python берет хэш ключа: `hash(key) -> index`.

Если бы объект был **изменяемым** (как список), и мы его изменили (добавили элемент), его хэш изменился бы.
1.  Положили данные по адресу, рассчитанному для `[1, 2]`.
2.  Сделали `append(3)`. Список стал `[1, 2, 3]`.
3.  Хэш изменился. Python ищет данные по новому адресу и **ничего не находит**.

Именно поэтому Python **запрещает** использовать изменяемые типы (`list`, `dict`, `set`) как хэшируемые.

---

### 4. Хитрый вопрос с собеседования (The "Gotcha")

*"Может ли кортеж (tuple) быть ключом словаря?"*
Ответ: **Зависит от того, что внутри.**

**RU:**
Кортеж сам по себе `immutable`. Но он может хранить ссылки на `mutable` объекты (например, списки).
Если кортеж содержит изменяемый объект, то сам кортеж **теряет свойство hashable**.

**EN:**
A tuple itself is `immutable`. However, it can hold references to `mutable` objects (like lists).
If a tuple contains a mutable object, the tuple **stops being hashable**.

```python
# 1. Простой кортеж - ОК
key_ok = (1, 2, "hello")
print(hash(key_ok))  # Работает
my_dict = {key_ok: "Value"} # ОК

# 2. Кортеж со списком внутри - ОШИБКА
key_bad = (1, 2, [3, 4]) 

# Кортеж всё еще immutable (мы не можем заменить список на что-то другое),
# НО список внутри можно менять (key_bad[2].append(5)).
# Поэтому Python говорит: "Я не могу гарантировать, что хэш будет вечным".

try:
    print(hash(key_bad))
except TypeError as e:
    print(e) 
    # Output: unhashable type: 'list'
```

---

### 5. Пользовательские классы / Custom Classes

Как ведут себя ваши собственные объекты?

**По умолчанию:**
*   Все экземпляры пользовательских классов **Mutable** (вы можете менять `self.x`).
*   НО они **Hashable**!
    *   `hash(obj)` вычисляется на основе `id(obj)` (адреса в памяти).
    *   `obj1 == obj2` только если это один и тот же объект (`is`).

**Ловушка:**
Если вы переопределите метод `__eq__` (чтобы сравнивать объекты по полям, как в DTO), то Python **обнулит** метод `__hash__` (сделает его `None`). Объект перестанет быть хэшируемым.

Вам придется реализовать `__hash__` вручную.

```python
class User:
    def __init__(self, uid, name):
        self.uid = uid
        self.name = name

    def __eq__(self, other):
        return self.uid == other.uid

    # Без этого метода объект User нельзя положить в set или dict key!
    def __hash__(self):
        return hash(self.uid)
```

---

### Резюме / Summary

1.  **Immutable:** Нельзя изменить содержимое (создается копия). Пример: `str`, `tuple`.
2.  **Mutable:** Можно менять на лету. Пример: `list`, `dict`.
3.  **Hashable:** Имеет стабильный хэш. Нужен для ключей словарей.
4.  **Связь:** Почти все `Immutable` — `Hashable`. Все `Mutable` — НЕ `Hashable` (стандартные типы).
5.  **Исключение:** Кортеж, содержащий список, **не хэшируем**.
