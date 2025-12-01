## Что такое паттерн "Одиночка"? / What is the Singleton Pattern?

**RU:**
**Singleton (Одиночка)** — это порождающий паттерн проектирования, который гарантирует, что у класса есть **только один экземпляр**, и предоставляет к нему глобальную точку доступа.

Простыми словами: сколько бы раз вы ни пытались создать объект этого класса, вы всегда будете получать ссылку на один и тот же объект в памяти.

**EN:**
**Singleton** is a creational design pattern that ensures a class has **only one instance**, while providing a global access point to this instance.

In simple terms: no matter how many times you try to create an object of this class, you will always get a reference to the exact same object in memory.

---

### 1. Зачем это нужно? / Why Do We Need It?

**RU:**
В контексте AutoQA и разработки, Singleton используется для управления общими ресурсами:
1.  **Конфигурация (Config):** Загрузили настройки один раз и используем везде. Не нужно читать файл настроек в каждом тесте.
2.  **Логгер (Logger):** Все части программы должны писать в один и тот же лог-файл, а не открывать его заново.
3.  **WebDriver:** В некоторых подходах используется один экземпляр браузера на всю сессию тестов.
4.  **База данных:** Одно подключение (Pool), которое переиспользуется.

**EN:**
In the context of AutoQA and development, Singleton is used to manage shared resources:
1.  **Configuration:** Load settings once and use them everywhere. No need to read the config file in every test.
2.  **Logger:** All parts of the program should write to the same log file instead of reopening it.
3.  **WebDriver:** Some approaches use a single browser instance for the entire test session.
4.  **Database:** A single connection (Pool) that is reused.

---

### 2. Реализации / Implementations

В Python есть несколько способов реализовать Singleton. Рассмотрим эволюцию от простого к правильному.

#### 2.1. Способ 1: Через метод `__new__` (Классический) / Method 1: Using `__new__` (Classic)

**RU:**
Мы переопределяем метод `__new__` (который отвечает за создание объекта), чтобы он сохранял созданный экземпляр и возвращал его при последующих вызовах.

**EN:**
We override the `__new__` method (responsible for object creation) to save the created instance and return it on subsequent calls.

```python
class Singleton:
    _instance = None  # Приватное поле для хранения экземпляра

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            # Если объекта нет, создаем его через родительский метод
            print("Creating the object...")
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, value):
        # ВНИМАНИЕ: __init__ вызывается каждый раз при обращении к классу!
        self.value = value

# Проверка / Check:
s1 = Singleton("First")
s2 = Singleton("Second")

print(s1 is s2)       # True (это один и тот же объект)
print(s1.value)       # "Second" (перезаписалось, так как __init__ сработал снова!)
```

**Минус:** Метод `__init__` вызывается каждый раз при `Singleton()`. Это может привести к сбросу атрибутов.

---

#### 2.2. Способ 2: Через Метакласс (Продвинутый) / Method 2: Using Metaclass (Advanced)

**RU:**
Это наиболее "правильный" и надежный способ в Python. Мы выносим логику создания экземпляра на уровень метакласса (класса, который создает другие классы). Это позволяет избежать повторного вызова `__init__`.

**EN:**
This is the most "correct" and robust way in Python. We move the instantiation logic to the metaclass level (the class that creates other classes). This avoids re-calling `__init__`.

```python
# 1. Создаем метакласс
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        # Этот метод срабатывает, когда мы пишем MyClass()
        if cls not in cls._instances:
            instance = super().__call__(*args, **kwargs)
            cls._instances[cls] = instance
        return cls._instances[cls]

# 2. Используем его (Python 3 syntax)
class DatabaseConnection(metaclass=SingletonMeta):
    def __init__(self):
        print("Connecting to DB...")
        self.status = "Connected"

# Проверка / Check:
db1 = DatabaseConnection() # Вывод: Connecting to DB...
db2 = DatabaseConnection() # (Ничего не выводит, __init__ не вызывается)

print(db1 is db2)      # True
print(db1.status)      # Connected
```

---

#### 2.3. Способ 3: Pythonic Way (Модуль) / Method 3: Pythonic Way (Module)

**RU:**
В Python модули **уже являются синглтонами**. При первом импорте модуль выполняется и кэшируется в `sys.modules`. При повторном импорте Python просто возвращает готовый объект модуля. Часто классы вообще не нужны.

**EN:**
In Python, modules **are already singletons**. On the first import, the module is executed and cached in `sys.modules`. On subsequent imports, Python simply returns the ready-made module object. Often, classes are not needed at all.

**Файл `config.py`:**
```python
# Это выполняется один раз / Executed once
print("Loading configuration...")
settings = {
    "db": "postgres",
    "timeout": 30
}
```

**Файл `main.py`:**
```python
import config # Вывод: Loading configuration...
import config # (Тишина)

print(config.settings["db"])
```

---

### 3. Анти-паттерн и риски / Anti-Patterns and Risks

**RU:**
Хотя Singleton популярен, злоупотреблять им в AutoQA опасно:
*   **Глобальное состояние (Global State):** Если один тест изменит состояние синглтона (например, поменяет настройки пользователя), это может сломать следующий тест.
*   **Сложность параллелизации:** При запуске тестов в несколько потоков (xdist) синглтон может стать узким местом или вызывать гонки данных (race conditions).

**EN:**
Although Singleton is popular, overusing it in AutoQA is risky:
*   **Global State:** If one test changes the state of the singleton (e.g., changes user settings), it might break the next test.
*   **Parallelization Issues:** When running tests in multiple threads (xdist), a singleton can become a bottleneck or cause race conditions.

### Резюме / Summary

| Способ / Method | Сложность / Complexity | Надежность / Reliability | Вердикт / Verdict |
| :--- | :--- | :--- | :--- |
| **Module** | ⭐ (Low) | ⭐⭐⭐⭐⭐ | Лучший выбор для простых данных (конфиги). |
| **`__new__`** | ⭐⭐ (Medium) | ⭐⭐⭐ | Просто, но `__init__` вызывается лишний раз. |
| **Metaclass** | ⭐⭐⭐⭐ (High) | ⭐⭐⭐⭐⭐ | Самый профессиональный способ для классов. |
