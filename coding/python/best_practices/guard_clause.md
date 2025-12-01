## Что такое GUARD CLAUSE? / What is a GUARD CLAUSE?

**RU:**
**Guard Clause** (защитное условие) — это техника написания кода, при которой мы проверяем граничные случаи и ошибки в самом начале функции и немедленно выходим из неё (`return` или `raise`), если что-то пошло не так.

Суть принципа: **"Fail Fast, Return Early"** (Падай быстро, возвращай управление рано). Вместо того чтобы оборачивать основной код в огромный блок `if`, мы избавляемся от "плохих" сценариев сразу.

**EN:**
A **Guard Clause** is a coding pattern where you validate edge cases and errors at the very beginning of a function and return immediately (`return` or `raise`) if requirements aren't met.

The core principle is: **"Fail Fast, Return Early"**. Instead of wrapping your main logic inside a huge `if` block, you eliminate the "bad" scenarios upfront.

---

### 1. Зачем это нужно? / Why Do We Need It?

**RU:**
Главная цель — избежать **"Arrow Code"** (Стреловидного кода) или "Пирамиды обреченности", когда вложенные `if` уходят далеко вправо.

**Преимущества:**
1.  **Читаемость:** Код становится "плоским" и линейным. Вы видите проверки, а затем основную логику.
2.  **Снижение когнитивной нагрузки:** Вам не нужно держать в голове контекст трех вложенных `if`, чтобы понять, что происходит в строке 20.
3.  **Чистота diff-ов:** Если вы добавите новую проверку, вы добавите две строчки сверху, а не будете сдвигать весь код функции вправо.

**EN:**
The main goal is to avoid **"Arrow Code"** or the "Pyramid of Doom," where nested `if` statements drift far to the right.

**Advantages:**
1.  **Readability:** The code becomes "flat" and linear. You see the checks, then the main logic.
2.  **Reduced Cognitive Load:** You don't need to keep the context of three nested `if`s in mind to understand line 20.
3.  **Cleaner Diffs:** Adding a new check means adding two lines at the top, rather than indenting the entire function body to the right.

---

### 2. Примеры: До и После / Examples: Before vs After

Давайте посмотрим на функцию обработки платежа.

#### До: Вложенный ад / Before: Nested Hell

**RU:**
Здесь основной код (списание денег) спрятан глубоко внутри. Чтобы добраться до него глазами, нужно прочитать все условия.

**EN:**
Here, the main logic (deducting money) is buried deep inside. To reach it visually, you have to parse all the conditions.

```python
def process_payment(user, amount):
    if user.is_authenticated:
        if amount > 0:
            if user.balance >= amount:
                # === HAPPY PATH ===
                user.balance -= amount
                user.save()
                print("Payment successful")
                # ==================
            else:
                print("Error: Insufficient funds")
        else:
            print("Error: Invalid amount")
    else:
        print("Error: User not logged in")
```

#### После: Guard Clauses / After: Guard Clauses

**RU:**
Мы инвертируем условия. Если что-то не так — выходим. Основная логика остается без отступов в конце функции.

**EN:**
We invert the conditions. If something is wrong — we exit. The main logic remains unindented at the bottom of the function.

```python
def process_payment(user, amount):
    # 1. Защита: Пользователь не авторизован
    if not user.is_authenticated:
        print("Error: User not logged in")
        return

    # 2. Защита: Некорректная сумма
    if amount <= 0:
        print("Error: Invalid amount")
        return

    # 3. Защита: Недостаточно средств
    if user.balance < amount:
        print("Error: Insufficient funds")
        return

    # === HAPPY PATH ===
    # Код стал линейным и понятным / Code is now linear and clear
    user.balance -= amount
    user.save()
    print("Payment successful")
```

---

### 3. Guard Clause в AutoQA / Guard Clause in AutoQA

**RU:**
В тестах этот паттерн часто используется в фикстурах или вспомогательных методах (helpers).

Например, функция, которая кликает по элементу, только если он видим:

**EN:**
In testing, this pattern is often used in fixtures or helper methods.

For example, a function that clicks an element only if it is visible:

```python
def safe_click(driver, locator):
    element = driver.find_element(*locator)
    
    # Guard Clause: Если элемент не видим, кликать нет смысла
    if not element.is_displayed():
        raise Exception(f"Element {locator} is not visible, cannot click!")

    # Guard Clause: Если элемент заблокирован (disabled)
    if not element.is_enabled():
        raise Exception(f"Element {locator} is disabled!")

    # Happy Path
    element.click()
```

### 4. Резюме / Summary

Если видите конструкцию вида:
```python
if condition:
    # много кода
```
Попробуйте переписать её как:
```python
if not condition:
    return
# много кода
```
