## Что такое Type Hinting? / What is Type Hinting?

**RU:**
**Type Hinting** — это способ явно указать, какой тип данных ожидается в переменной, аргументе функции или возвращаемом значении.

**Важно:** Python остается **динамически типизированным** языком. Интерпретатор **игнорирует** ваши подсказки при запуске (runtime). Если вы напишете `x: int = "hello"`, программа запустится и будет работать, пока не упадет с логической ошибкой.
Типы нужны не для Python, а для:
1.  **Вас** (читаемость кода).
2.  **IDE** (автодополнение и подсказки).
3.  **Статических анализаторов** (mypy, Pylance), которые найдут ошибки *до* запуска.

**EN:**
**Type Hinting** is a way to explicitly specify the expected data type of a variable, function argument, or return value.

**Important:** Python remains a **dynamically typed** language. The interpreter **ignores** your hints at runtime. If you write `x: int = "hello"`, the program will run until it crashes with a logic error.
Types are not for Python, but for:
1.  **You** (code readability).
2.  **IDEs** (autocomplete and hints).
3.  **Static Analyzers** (mypy, Pylance) to catch errors *before* execution.

---

### 1. Базовый синтаксис / Basic Syntax

```python
# 1. Переменные / Variables
# name: type = value
age: int = 25
name: str = "Alice"
scores: list[int] = [10, 20, 30]  # Python 3.9+ (не нужен import List)

# 2. Функции / Functions
# def func(arg: type) -> return_type:
def greet(name: str) -> str:
    return f"Hello, {name}"

def connect(retry: bool = False) -> None:
    pass
```

---

### 2. Специальные типы / Special Types

Эти типы делают аннотации гибкими. Большинство лежит в модуле `typing` (или встроенном синтаксисе в новых версиях).

#### 2.1. `Any` (Избегайте его! / Avoid it!)
**RU:** Означает "что угодно". Использование `Any` отключает проверки типов. Используйте только в крайнем случае, когда тип действительно может быть любым (например, `print()`).

**EN:** Means "anything". Using `Any` disables type checking. Use only as a last resort when the type truly can be anything.

```python
from typing import Any

def parse_json(data: Any) -> Any:
    ...
```

#### 2.2. `Union` и `|` (Или то, или это)
**RU:** Когда переменная может быть одного из нескольких типов.

**EN:** When a variable can be one of several types.

```python
# Python 3.10+ (Best Practice)
def process_id(user_id: int | str) -> None:
    pass

# Python < 3.10
from typing import Union
def process_id(user_id: Union[int, str]) -> None:
    pass
```

#### 2.3. `Optional` и `| None`
**RU:** Если переменная может быть `None`.

**EN:** If a variable can be `None`.

```python
# Python 3.10+
def get_user(id: int) -> str | None:
    return "User" if id > 0 else None
```

#### 2.4. `Literal` (Конкретные значения)
**RU:** Очень полезно в AutoQA! Указывает, что переменная может принимать только конкретные *значения*, а не просто любой `str` или `int`.

**EN:** Very useful in AutoQA! Specifies that a variable can only take specific *values*, not just any `str` or `int`.

```python
from typing import Literal

def open_browser(browser_name: Literal["chrome", "firefox", "safari"]):
    ...

open_browser("chrome")  # OK
# open_browser("edge")  # Mypy Error: "edge" not allowed
```

---

### 3. Продвинутые фишки (Modern Python) / Advanced Features

#### 3.1. `Self` (Python 3.11+)
Идеально для паттерна **Page Object** и методов, возвращающих `self` (Fluent Interface / Builder Pattern). Раньше приходилось писать сложные конструкции с TypeVar.

```python
from typing import Self

class LoginPage:
    def enter_username(self, name: str) -> Self:
        # ... logic ...
        return self

    def enter_password(self, pwd: str) -> Self:
        # ... logic ...
        return self

# Цепочка вызовов работает с автокомплитом!
LoginPage().enter_username("admin").enter_password("123")
```

#### 3.2. `TypedDict` (Типизированные словари)
Когда вам нужен словарь с конкретной структурой (например, JSON Payload), но создавать `dataclass` лень или нельзя.

```python
from typing import TypedDict

class UserPayload(TypedDict):
    name: str
    age: int
    role: Literal["admin", "guest"]

# IDE подскажет ключи!
data: UserPayload = {"name": "Alice", "age": 30, "role": "admin"}
```

#### 3.3. `override` (Python 3.12+)
Декоратор, который помогает избежать ошибок при наследовании. Если вы опечатались в названии метода родителя, статический анализатор выдаст ошибку.

```python
from typing import override

class BasePage:
    def get_url(self) -> str:
        return "http://base"

class LoginPage(BasePage):
    @override
    def get_url(self) -> str:  # Если в BasePage переименуют этот метод, здесь будет ошибка
        return "http://login"
```

---

### 4. Generics (Обобщения) — `TypeVar`

**RU:**
Нужны, когда функция работает с разными типами, но должна возвращать *тот же* тип, который получила на вход. Без дженериков мы бы потеряли информацию о типе.

**EN:**
Needed when a function works with different types but must return the *same* type it received as input. Without generics, we would lose type information.

#### Классический синтаксис (до 3.12):
```python
from typing import TypeVar, List

T = TypeVar("T")  # Объявляем переменную типа

def get_first(items: list[T]) -> T:
    return items[0]

# IDE понимает типы:
x = get_first([1, 2, 3])      # x: int
y = get_first(["a", "b"])     # y: str
```

#### Новый синтаксис (Python 3.12+):
Теперь дженерики встроены в синтаксис функций и классов!

```python
# <T> указывается в квадратных скобках перед аргументами
def get_first[T](items: list[T]) -> T:
    return items[0]

class Box[T]:
    def __init__(self, item: T):
        self.item = item
```

---

### 5. Под капотом / Under the Hood

Куда деваются эти типы при запуске?
Они сохраняются в специальном атрибуте `__annotations__` у функций и классов.

```python
def add(x: int, y: int) -> int:
    return x + y

print(add.__annotations__)
# {'x': <class 'int'>, 'y': <class 'int'>, 'return': <class 'int'>}
```

Библиотеки вроде **Pydantic** или **FastAPI** используют это поле в Runtime, чтобы валидировать данные. Но сам Python (интерпретатор) это поле не использует.

---

### 6. Зачем это в AutoQA? / Use Cases in AutoQA

1.  **API Клиенты:**
    Вы описываете методы API с типами. Тесты пишутся быстрее, потому что IDE подсказывает, какие поля есть в ответе (`response.data.user_id`), вместо того чтобы гадать и смотреть в Swagger.
2.  **Page Objects:**
    Методы возвращают другие Page Objects.
    ```python
    def click_login(self) -> "DashboardPage":
        self.btn.click()
        return DashboardPage(self.driver)
    ```
    Это дает возможность писать тесты цепочками: `login_page.login().open_settings()...`
3.  **Безопасность:**
    `Literal` помогает не ошибиться в строковых константах (например, статусы заказа "PENDING", "DONE").
