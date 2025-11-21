## Что такое магические методы? / What is the "magic" (dunder) methods?

**RU:**
Магические методы (или dunder-методы) — это специальные методы с двойными подчеркиваниями по бокам (например, `__init__`). Они позволяют вашим объектам использовать встроенные возможности Python: их можно будет сравнивать (`==`), получать длину (`len()`), использовать в циклах (`for`) и многое другое.

Они вызываются не напрямую, а автоматически, когда вы используете соответствующий синтаксис языка.

**EN:**
Magic methods (or dunder methods) are special methods with double underscores (e.g., `__init__`). They allow your custom objects to use Python's built-in features: you can compare them (`==`), get their length (`len()`), use them in loops (`for`), and much more.

They aren't called directly but are invoked automatically when you use the corresponding language syntax.

---

### 1. Создание и представление объекта/Creation and Representation

#### `__new__(cls, ...)`
**RU:** Первым вызывается при создании объекта. Его задача — **создать и вернуть** новый экземпляр класса. Используется редко, в основном для продвинутых техник, таких как реализация синглтонов или создание классов, наследуемых от неизменяемых (immutable) типов.

**EN:** The first method called when creating an object. Its job is to **create and return** a new instance of the class. It is used rarely, mainly for advanced techniques like implementing singletons or subclassing immutable types.

**Применение / Use Case:**
Контроль над процессом создания экземпляра. `__new__` решает, будет ли создан новый объект или будет возвращен уже существующий.

```python
class Singleton:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            print("Создание нового экземпляра")
            cls._instance = super().__new__(cls)
        return cls._instance

a = Singleton() # -> Создание нового экземпляра
b = Singleton() # -> (ничего не печатает, возвращается старый экземпляр)
```

#### `__init__(self, ...)`
**RU:** Конструктор объекта. Вызывается **после `__new__`** для **настройки (инициализации)** уже созданного экземпляра.

**EN:** The object constructor. It's called **after `__new__`** to **configure (initialize)** the newly created instance.

**Применение / Use Case:**
Используется для задания начального состояния объекта при его создании. Внутри этого метода определяются атрибуты экземпляра (например, `self.name = name`).

```python
class User:
    def __init__(self, user_id, name):
        self.id = user_id
        self.name = name
```

#### `__str__(self)` и `__repr__(self)`

*   **`__str__`**: Возвращает **человекочитаемое** строковое представление. Срабатывает при вызове `print(obj)` или `str(obj)`.
*   **`__repr__`**: Возвращает **техническое**, однозначное представление для разработчика. В идеале, его результат можно скопировать и выполнить для воссоздания объекта.

**Применение / Use Case:**
Ключевые методы для отладки и логирования. Позволяют контролировать, как объект будет отображаться в виде строки, что делает вывод программ более информативным.

```python
class User:
    def __init__(self, user_id, name):
        self.id = user_id
        self.name = name

    def __str__(self):
        return f"Пользователь {self.name}"

    def __repr__(self):
        return f"User(user_id={self.id!r}, name={self.name!r})"

user = User(123, "test_user")
print(user)       # вызовет __str__ -> Пользователь test_user
print(repr(user)) # вызовет __repr__ -> User(user_id=123, name='test_user')
```

---

### 2. Эмуляция контейнеров/Behaving Like a Container

#### `__len__(self)` и `__getitem__(self, key)`
*   `__len__`: Позволяет использовать функцию `len()` с вашим объектом.
*   `__getitem__`: Позволяет получать элементы по индексу или ключу (`obj[0]`).

**Применение / Use Case:**
Эти методы позволяют вашему классу вести себя как стандартный контейнер (например, список). `__len__` добавляет поддержку `len()`, а `__getitem__` — доступ по индексу и итерацию (`for item in obj`).

```python
class MyCollection:
    def __init__(self, items):
        self._items = list(items)

    def __len__(self):
        return len(self._items)

    def __getitem__(self, index):
        return self._items[index]

collection = MyCollection(["a", "b", "c"])
print(f"Длина: {len(collection)}") # -> Длина: 3
print(f"Первый элемент: {collection[0]}") # -> Первый элемент: a
for item in collection:
    print(item) # работает благодаря __getitem__
```

---

### 3. Сравнение и логика/Comparison and Logic

#### `__eq__(self, other)`
**RU:** Определяет поведение оператора `==` (равно).

**EN:** Defines the behavior of the `==` (equality) operator.

**Применение / Use Case:**
Определяет логику проверки на равенство. По умолчанию Python сравнивает адреса объектов в памяти. Этот метод позволяет сравнивать объекты по их внутренним данным (например, по ID).

#### `__hash__(self)`
**RU:** Позволяет объекту быть "хэшируемым", то есть использоваться в качестве ключа в словаре (`dict`) или элемента в множестве (`set`). Должен возвращать целое число.

**EN:** Allows an object to be "hashable," meaning it can be used as a key in a `dict` or an element in a `set`. Must return an integer.

**Правило:** Если `a == b`, то обязательно `hash(a) == hash(b)`. Поэтому, реализуя `__eq__`, часто нужно реализовать и `__hash__`.

```python
class User:
    def __init__(self, user_id, name):
        self.id = user_id
        self.name = name

    def __eq__(self, other):
        if not isinstance(other, User):
            return NotImplemented
        return self.id == other.id

    def __hash__(self):
        # Хэшируем по уникальному и неизменяемому полю
        return hash(self.id)

user1 = User(123, "Alice")
user2 = User(123, "Alice")
user3 = User(456, "Bob")

print(user1 == user2) # True

# Теперь объекты можно хранить в set или использовать как ключи dict
user_set = {user1, user2, user3}
print(len(user_set)) # -> 2
```

#### `__bool__(self)`
**RU:** Определяет, является ли объект "истинным" или "ложным" в логическом контексте (например, `if my_obj:`).

**EN:** Defines if an object is "truthy" or "falsy" in a boolean context (e.g., `if my_obj:`).

**Применение / Use Case:**
Определяет "истинность" объекта в логических выражениях (`if obj:`). Полезно для быстрой проверки, содержит ли контейнер данные или находится ли объект в "активном" состоянии.

```python
class MyCollection:
    # ... (код из примера выше) ...
    def __bool__(self):
        return len(self._items) > 0

empty_collection = MyCollection([])
if not empty_collection:
    print("Коллекция пуста") # -> Коллекция пуста
```

---

### 4. Управление ресурсами (Контекстные менеджеры)/Resource Management (Context Managers)

#### `__enter__(self)` и `__exit__(self, exc_type, exc_val, exc_tb)`
**RU:** Позволяют использовать объект с оператором `with`. `__enter__` выполняется в начале блока, а `__exit__` — в конце, даже если внутри произошло исключение.

**EN:** Allow an object to be used with the `with` statement. `__enter__` is executed at the start of the block, and `__exit__` is executed at the end, even if an exception occurs.

**Применение / Use Case:**
Реализуют протокол менеджера контекста. Это гарантирует, что ресурсы (файлы, сетевые соединения) будут корректно и безопасно освобождены.

```python
class ManagedFile:
    def __init__(self, filename):
        self._filename = filename

    def __enter__(self):
        print(f"Открываю файл {self._filename}")
        self._file = open(self._filename, 'w')
        return self._file

    def __exit__(self, exc_type, exc_value, traceback):
        if self._file:
            print(f"Закрываю файл {self._filename}")
            self._file.close()

with ManagedFile("hello.txt") as f:
    f.write("hello, world!")
# Файл автоматически закроется здесь
```

---

### 5. Вызов объекта как функции/Making an Object Callable

#### `__call__(self, *args, **kwargs)`
**RU:** Позволяет вызывать экземпляр класса так, как будто это функция.

**EN:** Allows an instance of a class to be called as if it were a function.

**Применение / Use Case:**
Удобно, когда объект представляет собой некое действие или "функцию с состоянием", то есть хранит какие-то данные между вызовами.

```python
class Adder:
    def __init__(self, value):
        self.value = value

    def __call__(self, x):
        return self.value + x

add_5 = Adder(5)
result = add_5(10) # Вызываем экземпляр как функцию
print(result) # -> 15
```
