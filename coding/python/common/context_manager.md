## Что такое Контекстный менеджер? / What is a Context Manager?

**RU:**
Контекстный менеджер — это объект, который управляет входом и выходом из **контекста выполнения**.
Синтаксически это оператор **`with`**.

Его главная задача — гарантировать, что ресурс (файл, соединение с БД, браузер) будет корректно освобожден (закрыт), даже если внутри блока кода произошла ошибка.

**EN:**
A Context Manager is an object that manages entering and exiting a **runtime context**.
Syntactically, it is the **`with`** statement.

Its main goal is to guarantee that a resource (file, DB connection, browser) is correctly released (closed), even if an error occurs inside the code block.

---

### 1. Зачем это нужно? / Why Do We Need It?

**Без менеджера (`try...finally`):**
Код громоздкий, легко забыть закрыть ресурс.

```python
file = open("log.txt", "w")
try:
    file.write("Hello")
    raise Exception("Oops!")
finally:
    file.close()  # Обязательно нужно закрыть!
```

**С менеджером (`with`):**
Код чистый, закрытие происходит автоматически.

```python
with open("log.txt", "w") as file:
    file.write("Hello")
    # При выходе из блока (даже с ошибкой) file.close() вызовется сам.
```

---

### 2. Под капотом: Протокол (`__enter__`, `__exit__`)

Чтобы класс работал с `with`, он должен реализовать два магических метода.

**RU:**
1.  **`__enter__(self)`**: Выполняется **до** начала блока. То, что этот метод вернет (`return`), попадет в переменную после `as`.
2.  **`__exit__(self, exc_type, exc_val, exc_tb)`**: Выполняется **после** выхода из блока. Сюда передается информация об исключении, если оно случилось.

**EN:**
1.  **`__enter__(self)`**: Executed **before** the block starts. Whatever this method returns becomes the variable after `as`.
2.  **`__exit__(self, exc_type, exc_val, exc_tb)`**: Executed **after** the block ends. Exception details are passed here if an error occurred.

#### Пример: Свой таймер / Example: Custom Timer

```python
import time

class Timer:
    def __enter__(self):
        self.start = time.time()
        return self  # Это попадет в `as t`

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.end = time.time()
        print(f"Time elapsed: {self.end - self.start:.4f} sec")
        # Если вернуть True, ошибка внутри блока будет подавлена (не упадет).
        # Если вернуть False (или None), ошибка полетит дальше.
        return False

# Использование
with Timer() as t:
    time.sleep(1)
    print("Doing work...")
# Output:
# Doing work...
# Time elapsed: 1.0005 sec
```

---

### 3. Модуль `contextlib` (The "Pythonic" Way)

Писать целый класс ради простого действия — долго.
Модуль **`contextlib`** позволяет превратить любую функцию-генератор в контекстный менеджер с помощью декоратора **`@contextmanager`**.

**Паттерн:**
1.  Код **до** `yield` = `__enter__`.
2.  Сам `yield` отдает значение.
3.  Код **после** `yield` = `__exit__`.

**Важно:** Обязательно используйте `try...finally`, чтобы код после `yield` выполнился даже при ошибке.

```python
from contextlib import contextmanager

@contextmanager
def temp_change_list(some_list, value):
    # 1. Setup (__enter__)
    print("Adding temporary item...")
    some_list.append(value)
    
    try:
        yield some_list  # Отдаем управление внутрь `with`
    finally:
        # 2. Teardown (__exit__)
        print("Removing item...")
        some_list.remove(value)

# Использование
my_list = [1, 2]
with temp_change_list(my_list, 3) as l:
    print(l)  # [1, 2, 3]
    # raise ValueError("Boom!") # Даже если тут упадет, элемент удалится

print(my_list) # [1, 2]
```

---

### 4. Асинхронные менеджеры (`async with`)

В современном AutoQA (Playwright, httpx) часто используются асинхронные ресурсы.
Для них существуют методы `__aenter__` и `__aexit__`.

```python
import aiohttp
import asyncio

async def main():
    # session.get(...) - это асинхронный контекстный менеджер
    async with aiohttp.ClientSession() as session:
        async with session.get('http://google.com') as resp:
            print(resp.status)

# asyncio.run(main())
```

Для создания своих используется `@asynccontextmanager` из `contextlib` (Python 3.7+).

---

### 5. Примеры для AutoQA / AutoQA Use Cases

#### 5.1. Временная смена конфига (Temporary Config)
Допустим, один тест требует выключенной валидации, а остальные — включенной. Менять глобально опасно (вдруг тест упадет и не вернет обратно?). `with` спасает ситуацию.

```python
@contextmanager
def disable_validation(config):
    old_value = config.validation_enabled
    config.validation_enabled = False
    try:
        yield
    finally:
        config.validation_enabled = old_value

# В тесте:
def test_something_risky(app_config):
    with disable_validation(app_config):
        # Тут валидация выключена
        app_config.do_something()
    # Тут она снова включена, что бы ни случилось внутри
```

#### 5.2. Соединение с базой данных (DB Connection)
Открываем транзакцию, делаем дела, а потом либо `commit`, либо `rollback` (если ошибка).

```python
@contextmanager
def db_session(connection_string):
    conn = create_connection(connection_string)
    cursor = conn.cursor()
    try:
        yield cursor
        conn.commit()  # Если всё ок — сохраняем
    except Exception:
        conn.rollback() # Если ошибка — отменяем
        raise
    finally:
        conn.close()    # Всегда закрываем соединение

# Использование
with db_session("postgres://...") as cursor:
    cursor.execute("INSERT INTO logs VALUES (1)")
```

#### 5.3. Мягкие проверки (Soft Assertions)
Вместо того чтобы падать при первой ошибке, мы собираем все ошибки и падаем в конце блока.

```python
@contextmanager
def soft_assert():
    errors = []
    
    # Мы возвращаем список, чтобы тест мог в него писать
    # В реальной жизни это будет объект-хелпер
    yield errors 
    
    if errors:
        raise AssertionError(f"Multiple failures: {errors}")

# Тест
with soft_assert() as verify:
    if 2 + 2 != 5:
        verify.append("Math is broken")
    if "a" not in "abc":
        verify.append("Char not found")

# Тут тест упадет и покажет сразу обе ошибки, если они были
```

---

### 6. Резюме / Summary

1.  **`with`** гарантирует выполнение очистки (Teardown).
2.  Под капотом это методы **`__enter__`** и **`__exit__`**.
3.  Для простоты используйте декоратор **`@contextmanager`** с конструкцией `try...finally`.
4.  В AutoQA это незаменимый инструмент для управления состоянием (Config, DB, Browser).
