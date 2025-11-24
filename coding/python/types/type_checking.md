## Что такое TYPE_CHECKING? / What is TYPE_CHECKING?

**RU:**
`TYPE_CHECKING` — это специальная константа из модуля `typing`.
Она равна `True`, когда ваш код проверяется статическим анализатором (например, mypy, Pylance или встроенным анализатором IDE), но всегда равна `False`, когда код запускается и работает (Runtime).

Это позволяет импортировать модули **только для подсказок типов**, не выполняя этот импорт при реальном запуске программы.

**EN:**
`TYPE_CHECKING` is a special constant from the `typing` module.
It is `True` when your code is being analyzed by a static type checker (like mypy, Pylance, or your IDE), but always `False` at runtime.

This allows you to import modules **only for type hinting** purposes without actually executing that import when the program runs.

---

### 1. Зачем это нужно? / Why Do We Need It?

**RU:**
Основных причин две:
1.  **Решение проблемы циклических импортов (Circular Imports):** Когда модуль A импортирует модуль B, а модуль B импортирует модуль A (например, `User` имеет ссылку на `Profile`, а `Profile` ссылается на `User`), Python выдает ошибку `ImportError`. `TYPE_CHECKING` разрывает этот круг.
2.  **Производительность:** Если вы импортируете "тяжелые" библиотеки только ради аннотации типов, это замедляет запуск скрипта. С `TYPE_CHECKING` эти библиотеки не загружаются в память при запуске.

**EN:**
There are two main reasons:
1.  **Solving Circular Imports:** When Module A imports Module B, and Module B imports Module A (e.g., `User` references `Profile`, and `Profile` references `User`), Python raises an `ImportError`. `TYPE_CHECKING` breaks this cycle.
2.  **Performance:** If you import "heavy" libraries just for type annotations, it slows down script startup. With `TYPE_CHECKING`, these libraries are not loaded into memory at runtime.

---

### 2. Синтаксис и использование / Syntax and Usage

**RU:**
Чтобы использовать импортированный класс внутри `TYPE_CHECKING`, его нужно указывать в кавычках (как строку) или использовать `from __future__ import annotations` (рекомендуемый современный способ).

**EN:**
To use a class imported inside `TYPE_CHECKING`, you must either quote it (as a string) or use `from __future__ import annotations` (the recommended modern way).

#### 2.1. Классический способ (строковые ссылки) / Classic way (string forward references)

```python
from typing import TYPE_CHECKING

# Этот блок выполняется ТОЛЬКО при проверке типов IDE/mypy
if TYPE_CHECKING:
    from database import DatabaseConnection  # Допустим, это тяжелый класс

class DataManager:
    # Мы используем кавычки, потому что в runtime переменная DatabaseConnection не существует
    def __init__(self, db: "DatabaseConnection"):
        self.db = db
```

#### 2.2. Современный способ (Python 3.7+) / Modern way (Python 3.7+)

**RU:**
Добавив импорт `annotations`, можно писать типы без кавычек, даже если они импортированы под `TYPE_CHECKING`. Python откладывает оценку аннотаций.

**EN:**
By adding the `annotations` import, you can write types without quotes, even if they are imported under `TYPE_CHECKING`. Python postpones the evaluation of annotations.

```python
from __future__ import annotations  # Должно быть первой строкой!
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from pages.login_page import LoginPage
    from services.api_client import ApiClient

class BaseTest:
    # Кавычки больше не нужны / Quotes are no longer needed
    def set_page(self, page: LoginPage):
        self.page = page
        
    def set_api(self, api: ApiClient):
        self.api = api
```

---

### 3. Как это работает "под капотом"? / How It Works "Under the Hood"

**RU:**
1.  **Runtime (Запуск кода):** Интерпретатор Python видит `if TYPE_CHECKING:`. Так как в runtime эта константа `False`, интерпретатор **пропускает** весь блок внутри `if`. Импорт не происходит. Ошибки цикличности нет.
2.  **Static Analysis (Проверка типов):** IDE или mypy видят `if TYPE_CHECKING:`. Для них эта константа `True`. Они "заходят" внутрь, считывают импорт и понимают, что `LoginPage` — это класс из другого файла. Теперь они могут давать подсказки (автокомплит).

**EN:**
1.  **Runtime:** The Python interpreter encounters `if TYPE_CHECKING:`. Since this constant is `False` at runtime, the interpreter **skips** the entire block inside the `if`. The import does not happen. No circular dependency error occurs.
2.  **Static Analysis:** The IDE or mypy sees `if TYPE_CHECKING:`. For them, this constant is `True`. They "enter" the block, read the import, and understand that `LoginPage` is a class from another file. Now they can provide type hints (autocomplete).

---

### 4. Реальный пример: Циклический импорт / Real World Example: Circular Import

Представьте структуру: `User` создает `Post`, а `Post` должен знать своего автора (`User`).

**Файл `post.py`:**
```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    # Если бы мы сделали обычный import здесь, случилась бы ошибка,
    # так как user.py тоже импортирует post.py
    from user import User 

class Post:
    def __init__(self, title: str, author: User):
        self.title = title
        self.author = author
```

**Файл `user.py`:**
```python
# Здесь нам нужен настоящий импорт, чтобы создавать объекты Post
from post import Post 

class User:
    def create_post(self, title: str) -> Post:
        return Post(title=title, author=self)
```

---

### 5. Когда НЕ использовать? / When NOT to use it?

**RU:**
Не используйте `TYPE_CHECKING`, если импортируемый класс нужен вам **во время выполнения** (например, для наследования или проверки `isinstance`).

**EN:**
Do not use `TYPE_CHECKING` if you need the imported class **at runtime** (for example, for inheritance or `isinstance` checks).

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from other import BaseClass

# ОШИБКА! BaseClass не существует при запуске программы
# ERROR! BaseClass does not exist at runtime
class MyClass(BaseClass): 
    pass
```
