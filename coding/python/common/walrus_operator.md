## Что такое моржовый оператор? / What is the walrus operator?

**RU:**
Моржовый оператор ```:=``` появился в Python 3.8. Он позволяет одновременно присваивать значение переменной и возвращать это значение как результат выражения.
Проще говоря — можно «встроить» присваивание прямо в условие, цикл или другое выражение.

**EN:**
The walrus operator ```:=``` was introduced in Python 3.8. It allows you to assign and return a value within a single expression.
In other words — you can “inline” assignments inside conditions, loops, or other expressions.

**RU:**
**Почему это полезно?**
*   **Улучшение читаемости:** В некоторых случаях код становится более линейным и понятным, так как вычисление и проверка происходят в одном месте.
*   **Повышение эффективности:** Помогает избежать повторных вызовов одной и той же функции. Это особенно важно, если функция ресурсоемкая (например, делает сетевой запрос или сложные вычисления).
*   **Упрощение логики:** Делает код короче и выразительнее в специфичных сценариях, таких как циклы `while` или list comprehensions.

**EN:**
**Why is this useful?**
*   **Improved Readability:** In some cases, the code becomes more linear and understandable because the computation and the check happen in the same place.
*   **Increased Efficiency:** It helps to avoid calling the same function twice. This is especially important if the function is resource-intensive (e.g., making a network request or performing complex calculations).
*   **Simplified Logic:** It makes the code shorter and more expressive in specific scenarios like `while` loops or list comprehensions.

### 0. Ключевые особенности синтаксиса / Key Syntax Features

**RU:**
1.  **Это выражение, а не инструкция.** Главное отличие `:=` от `=` в том, что `:=` является частью *выражения* (expression), которое возвращает значение. А `=` — это *инструкция* (statement), которая ничего не возвращает. Поэтому нельзя написать `x := 5` на отдельной строке.

    ```python
    # Так работает
    x = 5

    # А так — нет, будет SyntaxError
    # x := 5
    ```

2.  **Почти всегда нужны круглые скобки.** Чтобы избежать синтаксической неоднозначности и улучшить читаемость, оператор `:=` почти всегда нужно заключать в скобки.

    ```python
    # Правильно
    if (n := len(data)) > 0:
        ...

    # Неправильно (SyntaxError или неверная логика)
    # if n := len(data) > 0:
    # Python попытается присвоить n результат (len(data) > 0), то есть True или False
    ```
    Исключение — когда оператор используется в условии верхнего уровня без сравнений, например, `if value := get_value():`.

**EN:**
1.  **It's an expression, not a statement.** The main difference between `:=` and `=` is that `:=` is part of an *expression* that returns a value. In contrast, `=` is a *statement* that returns nothing. This is why you cannot write `x := 5` on its own line.

    ```python
    # This works
    x = 5

    # This does not — it's a SyntaxError
    # x := 5
    ```

2.  **Parentheses are almost always required.** To avoid syntactical ambiguity and improve readability, the walrus operator must almost always be enclosed in parentheses.

    ```python
    # Correct
    if (n := len(data)) > 0:
        ...

    # Incorrect (SyntaxError or wrong logic)
    # if n := len(data) > 0:
    # Python would try to assign the result of (len(data) > 0) to n, which is True or False
    ```
    The exception is when the operator is used in a top-level condition without comparisons, like `if value := get_value():`.

### 0.1 Принцип работы ("под капотом") / How It Works ("Under the Hood")

**RU:**
Оператор работает в три простых шага в рамках одного выражения. Рассмотрим на примере: `if (n := len([1, 2, 3])) > 0:`

1.  **Вычисление:** Сначала Python вычисляет выражение справа от `:=`. В нашем случае `len([1, 2, 3])` превращается в `3`.
2.  **Присваивание:** Полученный результат (`3`) немедленно присваивается переменной слева. Теперь `n` равно `3`.
3.  **Возврат значения:** Всё выражение в скобках `(n := ...)` "возвращает" это же значение (`3`) для дальнейшего использования.

После этого Python продолжает выполнять внешнюю часть: `if 3 > 0:`. Условие истинно.

Краткая формула работы: **Вычислить → Присвоить → Вернуть значение.**

**EN:**
The operator works in three simple steps within a single expression. Let's use this example: `if (n := len([1, 2, 3])) > 0:`

1.  **Evaluation:** First, Python evaluates the expression to the right of `:=`. In our case, `len([1, 2, 3])` becomes `3`.
2.  **Assignment:** The resulting value (`3`) is immediately assigned to the variable on the left. Now `n` is equal to `3`.
3.  **Return Value:** The entire parenthesized expression `(n := ...)` then "returns" that same value (`3`) for further use.

After that, Python continues with the outer part: `if 3 > 0:`. The condition is true.

The short formula for how it works is: **Evaluate → Assign → Return the value.**


### 1. Базовые примеры / Basic Examples
```py
if (n := len(data)) > 0:
    print(f"There are {n} items in the list")
```

#### 1.2. Экономия одной строки / Saving a line of code

До/Before:
```py
value = input()
if value:
    print(value)

```
После/After:
```py
if value := input():
    print(value)
```

### 2. Примеры в циклах / Loop Examples

#### 2.1. Чтение строк пока не пустая / Read lines until empty
```py
while (line := input("Enter line: ")) != "":
    print("You typed:", line)
```
#### 2.2. Обработка данных в потоке / Stream processing
До/Before:
```py
while True:
    chunk = file.read(4096)
    if not chunk:
        break
    process(chunk)
```
После/After:
```py
while chunk := file.read(4096):
    process(chunk)
```

### 3. Реальные рабочие сценарии / Practical Real-World Use Cases

#### 3.1. Фильтрация данных с сохранением результата / Filtering while caching the result
```py
results = [item for item in data if (x := expensive_compute(item)) > 10]
```
Заметка: функция ```expensive_compute(item)``` вызывается только один раз
Note: function ```expensive_compute(item)``` calls only ones

#### 3.2. Парсинг текста / Text parsing
```py
import re

if match := re.search(r"\d+", text):
    print("Found a number:", match.group())
```

#### 3.3. Ленивая инициализация / Lazy initialization
```py
def get_config():
    return {"port": 8080}

if not (cfg := globals().get("CONFIG")):
    cfg = globals()["CONFIG"] = get_config()

print(cfg)
```

### 4. Более сложные примеры / Advanced Examples

#### 4.1. Морж внутри list comprehension / Walrus inside comprehensions
```py
filtered = [
    (item, value)
    for item in items
    if (value := compute(item)) > 0
]
```
#### 4.2. Сложные условия / Complex conditions
```py
if (user := db.find_user(id)) and (age := user.get_age()) > 18:
    print(f"User found: age {age}")
```

#### 4.3. Мини-парсер на моржах / Mini-parser using walrus operator
```py
tokens = []
while token := lexer.next():
    if token.type == "NUMBER" and (value := int(token.text)) > 100:
        tokens.append(("BIG_NUMBER", value))
```

### 5. Анти-паттерны / Anti-Patterns

#### 5.1. Не стоит злоупотреблять / Don’t overuse it
```py
if (x := func()) and (y := other()) and (z := more()):
    ...
```
Проблема: плохо читается
Problem: readability suffers

#### 5.2. Присваивание в выражениях, где это не ожидается
Плохо/Bad:
```py
print((n := len(data)))
```
Хорошо/Good:
```py
n = len(data)
print(n)
```

## Дополнительные ресурсы / Additional Resources

- [PEP 572 — The Walrus Operator](https://peps.python.org/pep-0572/)
- [Official Python Documentation](https://docs.python.org/3/whatsnew/3.8.html#assignment-expressions)
- [Real Python Article](https://realpython.com/python-walrus-operator/)
