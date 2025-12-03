## Что такое Dataclass? / What is a Dataclass?

**RU:**
**Dataclass** — это "синтаксический сахар", встроенный в Python (модуль `dataclasses`).
Это декоратор `@dataclass`, который **автоматически генерирует** скучный шаблонный код (boilerplate) для классов, предназначенных в основном для хранения данных.

Вам больше не нужно вручную писать методы `__init__`, `__repr__`, `__eq__` и `__hash__`. Вы просто описываете поля и их типы, а Python делает остальное за вас.

**EN:**
**Dataclass** is "syntactic sugar" built into Python (the `dataclasses` module).
It is a decorator `@dataclass` that **automatically generates** boring boilerplate code for classes designed primarily to store data.

You no longer need to manually write `__init__`, `__repr__`, `__eq__`, or `__hash__` methods. You simply describe the fields and their types, and Python does the rest.

---

### 1. До и После / Before vs After

**Классический класс (Old School):**
```python
class User:
    def __init__(self, id, name, is_admin=False):
        self.id = id
        self.name = name
        self.is_admin = is_admin

    def __repr__(self):
        return f"User(id={self.id}, name={self.name}, is_admin={self.is_admin})"

    def __eq__(self, other):
        if not isinstance(other, User):
            return False
        return self.id == other.id and self.name == other.name
```

**Dataclass (Modern):**
```python
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
    is_admin: bool = False
```
*Эти два куска кода делают абсолютно одно и то же. Но второй в 3 раза короче.*

---

### 2. Под капотом: Как это работает? / Under the Hood

Это не магия, а **генерация кода**.
Когда Python встречает декоратор `@dataclass`:
1.  Он сканирует аннотации типов внутри класса.
2.  Он **генерирует текст кода** для методов `__init__`, `__repr__` и т.д.
3.  Он выполняет этот код (`exec`) и добавляет полученные методы в ваш класс.
4.  Всё это происходит один раз — при загрузке класса (Import time), поэтому **производительность при создании объектов не страдает**.

---

### 3. Ключевые фишки (Python 3.10+) / Key Features

#### 3.1. Неизменяемость (`frozen=True`)
Делает объект похожим на `NamedTuple`. Вы не сможете изменить поля после создания. Это делает объект **хэшируемым** (можно использовать как ключ словаря).

```python
@dataclass(frozen=True)
class Config:
    url: str
    timeout: int

conf = Config("http://api.com", 30)
# conf.timeout = 60  # FrozenInstanceError
```

#### 3.2. Оптимизация памяти (`slots=True`) — New in 3.10
**Важно!** Обычные классы хранят атрибуты в словаре `__dict__`, что ест много памяти.
Параметр `slots=True` заставляет Python использовать `__slots__` (фиксированный массив памяти).
*   **Память:** Потребление снижается в 2-3 раза.
*   **Скорость:** Доступ к полям быстрее на ~20%.

```python
@dataclass(slots=True)
class Point:
    x: int
    y: int
```

---

### 4. Dataclass vs Остальные / Comparison

| Тип | Изменяемость | Валидация типов | Методы | Память | Использование |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Dict** | Да | Нет | Нет | Много | JSON, простые данные |
| **NamedTuple** | Нет | Только хинты | Да | Мало | Простые неизменяемые структуры |
| **Dataclass** | Да (можно `frozen`) | Только хинты | Да | Средне (со `slots`) | DTO, Бизнес-логика, Конфиги |
| **Pydantic** | Да | **Строгая (Runtime)** | Много | Больше | **API Parsing, Валидация JSON** |

**Важно про Pydantic:**
Dataclass **НЕ** проверяет типы данных в runtime. Если вы напишете `User(id="строка")`, он проглотит это (если не использовать Pydantic).
В AutoQA для **парсинга ответов API** лучше использовать **Pydantic**, а для внутренних структур тестов — **Dataclass**.

---

### 5. Ловушка для новичков: Mutable Defaults / The Mutable Default Trap

Это самая частая ошибка. Нельзя использовать изменяемые объекты (списки, словари) как значения по умолчанию напрямую.

**Ошибка (ValueError):**
```python
@dataclass
class Inventory:
    items: list = []  # Python запретит это!
```

**Правильно (`field` + `default_factory`):**
Нужно использовать фабрику, которая будет создавать *новый* список для каждого нового объекта.

```python
from dataclasses import dataclass, field

@dataclass
class Inventory:
    items: list[str] = field(default_factory=list)

inv1 = Inventory()
inv1.items.append("Apple")

inv2 = Inventory()
print(inv2.items) # [] (Пусто, как и должно быть!)
```

---

### 6. Зачем это в AutoQA? / Use Cases in AutoQA

#### 6.1. API Payloads (Тела запросов)
Вместо того чтобы собирать JSON из словарей вручную, мы создаем объекты.

```python
@dataclass
class CreateUserRequest:
    username: str
    email: str
    roles: list[str] = field(default_factory=lambda: ["user"])

payload = CreateUserRequest("test_user", "test@mail.com")
# Превращаем в словарь для отправки: 
# asdict(payload) -> {'username': '...', 'roles': ['user']}
```

#### 6.2. Test Data Objects (DTO)
Удобно передавать сложные данные между фикстурами и тестами.

```python
@dataclass(frozen=True)
class TestCredentials:
    valid_user: tuple[str, str]
    invalid_user: tuple[str, str]
    admin_token: str
```
