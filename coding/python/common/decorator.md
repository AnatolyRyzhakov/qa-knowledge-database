# Что такое Декоратор? / What is a Decorator?

**RU:**
**Декоратор** — это функция, которая принимает другую функцию и расширяет её поведение, не изменяя её исходный код.
Это паттерн "Обертка" (Wrapper).

**Аналогия:** Представьте подарок.
*   Функция — это сам подарок (например, книга).
*   Декоратор — это упаковочная бумага и бантик.
Вы получаете ту же самую книгу, но теперь она выглядит красивее и праздничнее. При этом текст книги внутри не изменился.

**EN:**
A **Decorator** is a function that takes another function and extends its behavior without modifying its source code.
It is essentially the "Wrapper" pattern.

**Analogy:** Imagine a gift.
*   The function is the gift itself (e.g., a book).
*   The decorator is the wrapping paper and the bow.
You still get the same book, but now it looks nicer and more festive. The content of the book inside remains unchanged.

---

### 1. Как это работает "под капотом"? / How It Works "Under the Hood"

Синтаксис с "собачкой" `@` — это просто "сахар" (Syntactic Sugar).

**RU:**
В Python функции — это объекты. Их можно передавать в другие функции как переменные.
Декоратор просто берет вашу функцию, кладет её внутрь другой функции (обертки) и возвращает эту обертку.

**EN:**
In Python, functions are objects. They can be passed to other functions as variables.
A decorator simply takes your function, puts it inside another function (the wrapper), and returns that wrapper.

#### Без сахара (The "Manual" Way):
```python
def my_decorator(func):
    def wrapper():
        print("Before...")
        func()
        print("After...")
    return wrapper

def say_hello():
    print("Hello!")

# Ручное декорирование / Manual decoration
say_hello = my_decorator(say_hello)

say_hello()
# Output:
# Before...
# Hello!
# After...
```

#### С сахаром (The `@` Syntax):
```python
@my_decorator
def say_hello():
    print("Hello!")
```
*Эти два примера делают абсолютно одно и то же.*

---

### 2. Как написать свой декоратор? / How to Write a Custom Decorator?

Стандартный шаблон декоратора выглядит так. Нам нужны `*args` и `**kwargs`, чтобы декоратор мог работать с любыми функциями (с аргументами или без).

**Шаблон:**
```python
import functools

def name_of_decorator(func):
    @functools.wraps(func)  # <--- ВАЖНО / IMPORTANT
    def wrapper(*args, **kwargs):
        # 1. Код ДО вызова функции
        print("Start...")
        
        # 2. Вызов оригинальной функции
        result = func(*args, **kwargs)
        
        # 3. Код ПОСЛЕ вызова функции
        print("End...")
        
        # 4. Возврат результата
        return result
    return wrapper
```

#### Зачем нужен `@functools.wraps`? / Why `@functools.wraps`?

**RU:**
Когда вы оборачиваете функцию `say_hello` в `wrapper`, Python начинает думать, что эта функция теперь называется `wrapper`. Она теряет своё оригинальное имя и документацию (`__doc__`).
`@functools.wraps(func)` копирует метаданные из оригинала в обертку. **Всегда используйте его!**

**EN:**
When you wrap `say_hello` inside `wrapper`, Python starts thinking the function is now named `wrapper`. It loses its original name and docstring (`__doc__`).
`@functools.wraps(func)` copies metadata from the original to the wrapper. **Always use it!**

---

### 3. Декораторы с аргументами / Decorators with Arguments

Иногда нам нужно передать настройки самому декоратору. Например: `@retry(times=5)`.
Для этого нужно добавить **еще один уровень вложенности**.

1.  Функция, принимающая аргументы (`times`).
2.  Функция, принимающая саму функцию (`func`).
3.  Функция-обертка (`wrapper`).

```python
import functools
import time

def retry(max_attempts=3, delay=1):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    print(f"Ошибка: {e}. Пробуем снова ({attempt + 1}/{max_attempts})...")
                    time.sleep(delay)
            # Если попытки кончились - поднимаем ошибку
            raise Exception("Все попытки исчерпаны")
        return wrapper
    return decorator

# Использование / Usage:
@retry(max_attempts=3, delay=0.5)
def flaky_api_call():
    import random
    if random.random() < 0.7:
        raise ValueError("Сбой сети")
    return "Success!"
```

---

### 4. Полезные примеры для AutoQA / Useful Examples for AutoQA

#### 4.1. Логирование шагов / Logging Steps
Автоматически логирует начало и конец теста или шага.

```python
import functools
import logging

def log_step(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        logging.info(f">>> STEP START: {func.__name__}")
        try:
            result = func(*args, **kwargs)
            logging.info(f"<<< STEP PASS: {func.__name__}")
            return result
        except Exception as e:
            logging.error(f"!!! STEP FAIL: {func.__name__} with error: {e}")
            raise # Пробрасываем ошибку дальше, чтобы тест упал
    return wrapper

@log_step
def click_login_button():
    print("Clicking...")
```

#### 4.2. Замер времени выполнения / Execution Timer
Полезно для поиска медленных тестов.

```python
import time
import functools

def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start_time = time.time()
        result = func(*args, **kwargs)
        end_time = time.time()
        print(f"Function {func.__name__} took {end_time - start_time:.4f} seconds")
        return result
    return wrapper
```

#### 4.3. Авторизация (Mock) / Auth Check
Пример из веб-разработки, но полезен и в тестах для проверки прав.

```python
def admin_required(func):
    @functools.wraps(func)
    def wrapper(user, *args, **kwargs):
        if not user.is_admin:
            raise PermissionError("User is not an admin!")
        return func(user, *args, **kwargs)
    return wrapper
```

---

### 5. Резюме / Summary

1.  **Декоратор** — это функция, которая оборачивает другую функцию.
2.  Используйте `*args` и `**kwargs` в `wrapper`, чтобы поддерживать любые аргументы.
3.  **Всегда** используйте `@functools.wraps(func)`, чтобы не ломать интроспекцию и отчеты (Allure, Pytest).
4.  Если декоратору нужны свои аргументы (`@decorator(arg)`), нужна тройная вложенность функций.
