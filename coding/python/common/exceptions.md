## Что такое исключения? / What is the exceptions?

**RU:**
Исключения (Exceptions) — это основной механизм обработки ошибок в Python. Когда в программе происходит что-то непредвиденное (например, деление на ноль или попытка открыть несуществующий файл), Python "выбрасывает" (raises) объект исключения.

Если это исключение не "поймать" (catch), программа аварийно завершится. Правильная обработка исключений позволяет создавать надежные и отказоустойчивые программы.

**EN:**
Exceptions are Python's primary mechanism for handling errors. When something unexpected occurs in a program (like dividing by zero or trying to open a file that doesn't exist), Python "raises" an exception object.

If this exception is not "caught," the program will crash. Proper exception handling allows you to build robust and fault-tolerant applications.

---

### 1. Зачем нужны исключения? / Why Do We Need Exceptions?

**RU:**
Исключения позволяют отделить логику обработки ошибок от основного "счастливого пути" (happy path) программы. Вместо того чтобы на каждом шагу проверять коды возврата (`if error: ...`), вы пишете чистый код в блоке `try`, а всю обработку ошибок выносите в блоки `except`.

**Ключевые преимущества:**
*   **Чистый код:** Основная логика не замусорена проверками ошибок.
*   **Централизованная обработка:** Ошибки из вложенных функций могут быть пойманы на более высоком уровне.
*   **Информативность:** Объект исключения содержит подробную информацию о том, что, где и почему пошло не так (traceback).

**EN:**
Exceptions allow you to separate error-handling logic from the main "happy path" of your program. Instead of checking return codes at every step (`if error: ...`), you write clean code in a `try` block and move all error handling to `except` blocks.

**Key advantages:**
*   **Cleaner Code:** The main logic is not cluttered with error checks.
*   **Centralized Handling:** Errors from nested functions can be caught at a higher level.
*   **Informativeness:** The exception object contains detailed information about what went wrong, where, and why (the traceback).

---

### 2. Синтаксис: Конструкция `try...except` / Syntax: The `try...except` Block

**RU:**
Основная конструкция для обработки исключений состоит из четырех ключевых слов:
*   `try`: Блок, в котором вы выполняете код, который **может** вызвать исключение.
*   `except`: Блок, который выполняется, **если** в блоке `try` произошло исключение указанного типа.
*   `else`: Блок, который выполняется, **если** в блоке `try` исключений **не было**.
*   `finally`: Блок, который выполняется **всегда** — независимо от того, было исключение или нет. Обычно используется для освобождения ресурсов (закрытия файлов, сетевых соединений).

**EN:**
The primary structure for handling exceptions consists of four keywords:
*   `try`: A block where you place code that **might** cause an exception.
*   `except`: A block that executes **if** an exception of a specified type occurs in the `try` block.
*   `else`: A block that executes **if** no exceptions occurred in the `try` block.
*   `finally`: A block that **always** executes, regardless of whether an exception occurred. It's typically used for cleanup actions (closing files, network connections).

```python
try:
    # 1. Попытка выполнить этот код
    result = 10 / int(input("Введите число: "))
except ValueError:
    # 2a. Выполняется, если введено не число (например, "abc")
    print("Ошибка: нужно ввести число!")
except ZeroDivisionError:
    # 2b. Выполняется, если введено число 0
    print("Ошибка: на ноль делить нельзя!")
else:
    # 3. Выполняется, если исключений не было
    print(f"Результат: {result}")
finally:
    # 4. Выполняется в любом случае
    print("Выполнение блока try-except завершено.")
```

---

### 3. Как это работает "под капотом"? / How It Works "Under the Hood"

**RU:**
1.  Python начинает выполнять код внутри блока `try`.
2.  Если возникает ошибка, выполнение `try` **немедленно прекращается**.
3.  Python создает объект-исключение и ищет подходящий блок `except` (сверху вниз).
4.  Если найден `except`, соответствующий типу исключения, выполняется его код, а затем код в блоке `finally`.
5.  Если подходящий `except` не найден, исключение "пробрасывается" на уровень выше по стеку вызовов. Если его так нигде и не поймают, программа завершается с ошибкой.

**EN:**
1.  Python starts executing the code inside the `try` block.
2.  If an error occurs, the execution of the `try` block **stops immediately**.
3.  Python creates an exception object and looks for a matching `except` block (from top to bottom).
4.  If an `except` block matching the exception type is found, its code is executed, followed by the code in the `finally` block.
5.  If no matching `except` is found, the exception is "propagated" up the call stack. If it's never caught, the program terminates with an error.

---

### 4. Основные встроенные исключения / Common Built-in Exceptions

| Исключение / Exception | RU: Когда возникает? | EN: When does it occur? |
| :--- | :--- | :--- |
| `ValueError` | Функция получает аргумент правильного типа, но некорректного значения. | A function receives an argument of the correct type but an inappropriate value. |
| `TypeError` | Операция или функция применяется к объекту неподходящего типа. | An operation or function is applied to an object of an inappropriate type. |
| `IndexError` | Индекс для последовательности (списка, кортежа) выходит за пределы диапазона. | A sequence subscript is out of range. |
| `KeyError` | Ключ не найден в словаре. | A dictionary key is not found. |
| `FileNotFoundError` | Попытка открыть несуществующий файл для чтения. | Trying to open a non-existent file for reading. |
| `AttributeError` | Попытка доступа к несуществующему атрибуту или методу объекта. | An attempt to access a non-existent attribute or method of an object. |
| `ZeroDivisionError` | Деление на ноль. | Division by zero. |

---

### 5. Как создать своё исключение / How to Create a Custom Exception

**RU:**
Создание собственных исключений — хорошая практика. Это делает ваш код более читаемым и позволяет ловить ошибки, специфичные для логики вашего приложения.

Чтобы создать кастомное исключение, достаточно унаследовать свой класс от встроенного класса `Exception`.

**EN:**
Creating your own exceptions is good practice. It makes your code more readable and allows you to catch errors specific to your application's logic.

To create a custom exception, simply inherit your class from the built-in `Exception` class.

```python
# 1. Создаем свой класс исключения
class PaymentError(Exception):
    """Исключение, связанное с обработкой платежей."""
    pass

# 2. Функция, которая может его "выбросить"
def process_payment(amount):
    if amount <= 0:
        # Используем `raise` для генерации исключения
        raise PaymentError("Сумма платежа должна быть положительной!")
    print(f"Платеж на сумму {amount} успешно обработан.")

# 3. Обрабатываем наше кастомное исключение
try:
    process_payment(-100)
except PaymentError as e:
    # `as e` позволяет получить доступ к объекту исключения
    print(f"Не удалось обработать платеж: {e}")

# Вывод:
# Не удалось обработать платеж: Сумма платежа должна быть положительной!
```
