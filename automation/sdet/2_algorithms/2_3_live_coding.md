# 📘 Глава 2. Алгоритмы и Live Coding (Подготовка к FAANG/BigTech)
## Тема 2.3. Паттерны Live Coding: Строки, Массивы, Математика и Стек

### Введение: кульминация подготовки к FAANG - секционированный Live Coding
На интервью в топовые компании от Senior SDET ожидают не просто решения задачи, а применения **паттернов проектирования алгоритмов** с оптимальной пространственно-временной сложностью. 

Более того, согласно стандартам качества **ISO/IEC 25010** (в частности, Security и Performance Efficiency), интервьюер будет жестко штрафовать за использование небезопасных встроенных функций (вроде `eval()`) или за аллокацию лишней памяти.

Ниже представлен разбор четырех классических паттернов с кодом, адаптированным под реалии алгоритмических собеседований 2025–2026 годов.

### Паттерн 1. Строки: Два Указателя (Two Pointers) на примере Палиндрома

**Задача:** Проверить, является ли строка палиндромом (читается одинаково слева направо и справа налево), игнорируя пробелы и регистр.

#### ❌ Bad Practice (Решение уровня Junior)
Написать `s == s[::-1]` в Python легко, но на интервью в Google или Amazon это вызовет вопросы. Срез `[::-1]` **создает полную копию строки в памяти**. Если вам на вход подадут лог-строку размером 1 GB, ваш код выделит еще 1 GB оперативной памяти (Space Complexity $O(n)$ ), что может привести к `OOM` (Out of Memory).

#### ✅ Best Practice: Паттерн "Два указателя" (Уровень Senior)
Используем два индекса: один идет с начала, другой с конца. Мы сравниваем символы на месте (in-place). Пространственная сложность: **$O(1)$**.

```python
def is_palindrome(s: str) -> bool:
    left, right = 0, len(s) - 1
    
    while left < right:
        # Пропускаем не буквенно-цифровые символы
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1
            
        # Сравниваем в нижнем регистре
        if s[left].lower() != s[right].lower():
            return False
            
        left += 1
        right -= 1
        
    return True
```

---

### Паттерн 2. Массивы: Поиск отсутствующего числа (XOR / Математика)

**Задача:** Дан массив из $n$ уникальных чисел в диапазоне от `0` до `n`. Одно число удалено. Найти его за время $O(n)$ и память $O(1)$.

#### ❌ Bad Practice
Сортировка массива ( $O(n \log n)$ ) или использование `HashSet` (дополнительная память $O(n)$) не удовлетворяют условиям задачи для Senior уровня.

#### ✅ Best Practice 1: Формула Гаусса
Мы вычисляем сумму арифметической прогрессии $\frac{N \times (N + 1)}{2}$ и вычитаем из нее фактическую сумму элементов массива. Разница и есть пропавшее число.

*Осторожно:* На языках со строгой типизацией (Java/C++) формула Гаусса может вызвать целочисленное переполнение (Integer Overflow) на очень больших массивах. 

#### ✅ Best Practice 2: Побитовый XOR (Идеальное решение)
Побитовое исключающее ИЛИ (XOR, оператор `^`) решает проблему переполнения. 

Свойства XOR: `a ^ a = 0` и `a ^ 0 = a`. Если мы применим XOR ко всем индексам и всем значениям, дубликаты уничтожатся, и останется только пропавшее число.

```python
def missing_number(nums: list[int]) -> int:
    missing = len(nums) # Инициализируем значением N
    
    for i, num in enumerate(nums):
        # XOR-им текущий результат с индексом и значением массива
        missing ^= i ^ num
        
    return missing
```

---

### Паттерн 3. Алгоритмическая Математика: Нули факториала

**Задача:** Найти количество замыкающих нулей (trailing zeroes) в факториале числа $N!$ (например, $5! = 120$ -> 1 нуль).

#### ❌ Bad Practice (Наивный подход)
Вычислять сам факториал, а затем делить его на 10. Факториал растет астрономически быстро ($100!$ имеет 158 цифр). Это приведет к колоссальным затратам памяти или переполнению типов (Overflow).

#### ✅ Best Practice: Подсчет множителей (Теорема Лежандра)
Нуль на конце числа образуется при умножении $2 \times 5$ . Поскольку в любом факториале двоек всегда больше, чем пятерок, количество нулей строго равно количеству **пятерок** в разложении числа на простые множители. Нам нужно просто поделить $N$ на $5$, затем на $25$, затем на $125$ и т.д., пока делитель не превысит $n$.

Сложность по времени: **$O(\log_5 n)$**, память: **$O(1)$** .

```python
def trailing_zeroes(n: int) -> int:
    zero_count = 0
    power_of_5 = 5
    
    # Считаем, сколько чисел делится на 5, затем на 25, 125 и т.д.
    while n >= power_of_5:
        zero_count += n // power_of_5
        power_of_5 *= 5
        
    return zero_count
```

---

### Паттерн 4. Парсинг строк: Калькулятор Логики (Stack)

**Задача:** Написать функцию, которая принимает строку с математическим выражением (например, `"3 + 2 * 2"`) и возвращает результат.

#### ❌ Bad Practice (Автоматический отказ на интервью)
Использование `eval("3 + 2 * 2")`. С точки зрения безопасности (Security Compliance), `eval()` позволяет выполнить произвольный вредоносный код (Code Injection). На интервью использование `eval()` запрещено.

#### ✅ Best Practice: Паттерн "Стек" (Stack Approach)
Используем стек (`list` в Python) для управления приоритетом операций (умножение/деление выполняются раньше сложения).
1. Мы читаем строку посимвольно.
2. Если видим число, формируем его.
3. Если видим знак (`+`, `-`, `*`, `/`), применяем *предыдущий* знак к собранному числу и кладем результат в стек.
4. В конце просто суммируем все элементы стека.
Сложность по времени: **$O(n)$**, память: **$O(n)$**.

```python
def calculate(s: str) -> int:
    stack =[]
    current_num = 0
    last_operator = '+' # По умолчанию первое число положительное
    
    # Добавляем фиктивный знак в конец, чтобы гарантированно обработать последнее число
    s += '+' 
    
    for char in s:
        if char.isdigit():
            # Собираем многозначные числа (например, '1' и '2' -> 12)
            current_num = current_num * 10 + int(char)
        elif char in "+-*/":
            # Применяем ПРЕДЫДУЩИЙ оператор к текущему числу
            if last_operator == '+':
                stack.append(current_num)
            elif last_operator == '-':
                stack.append(-current_num)
            elif last_operator == '*':
                # Извлекаем последний элемент, умножаем и кладем обратно
                stack.append(stack.pop() * current_num)
            elif last_operator == '/':
                # Для Python используем int() для отбрасывания дробной части (truncation toward zero)
                stack.append(int(stack.pop() / current_num))
            
            # Сбрасываем текущее число и обновляем оператор
            current_num = 0
            last_operator = char
            
    return sum(stack)
```
*(Для выражений со скобками `( )` используется рекурсивный вызов этой же функции или усложненный стек контекстов).*

---

### 💡 Рекомендации к Live Coding (The "Think Aloud" Protocol)
В FAANG (и современном BigTech в 2026 году) молчаливое написание правильного кода гарантирует вам отказ. От Senior SDET ожидают навыка коммуникации:
1. **Clarify (Уточните ограничения):** Прежде чем писать код калькулятора, спросите: *"Могут ли в строке быть отрицательные числа? Гарантируется ли, что выражение валидно? Есть ли пробелы?"*.
2. **Brute Force (Озвучьте решение "в лоб"):** Скажите: *"Я могу решить поиск числа через HashSet с памятью* $O(n)$".
3. **Optimize (Предложите оптимум):** *"Однако, чтобы сэкономить память, я применю побитовый XOR* ( $O(1)$ )".
4. **Edge Cases (Угловые случаи):** В конце всегда проговорите вслух проверку пустого массива `

Sources:
1. [Valid Palindrome - The Two-Pointer Trick 90% of People Miss (LeetCode 125)](https://www.youtube.com/watch?v=u5plzLM1HyI)
2. [I Solved 50+ Palindrome Problems on LeetCode and These 5 Methods Actually Changed How I Code](https://dev.to/emmimal_alexander_3be8cc7/i-solved-50-palindrome-problems-on-leetcode-and-these-5-methods-actually-changed-how-i-code-3b5b)
3. [125. Valid Palindrome - Explanation](https://neetcode.io/solutions/valid-palindrome)
4. [#125 Valid Palindrome : Intro to Two-Pointers](https://medium.com/@ayaan-javed/125-valid-palindrome-intro-to-two-pointers-b6df78ef4088)
5. [125. Valid Palindrome](https://algo.monster/liteproblems/125)
6. [268. Missing Number - Explanation](https://neetcode.io/solutions/missing-number)
7. [Leetcode Missing Number | Bit Manipulation | Python](https://www.youtube.com/watch?v=8YUYaGS5hqY)
8. [268. Missing Number](https://www.talentd.in/dsa-corner/questions/missing-number)
9. [Unmasking Bitmasked Dynamic Programming](https://medium.com/free-code-camp/unmasking-bitmasked-dynamic-programming-25669312b77b)
10. [172. Factorial Trailing Zeroes](https://algo.monster/liteproblems/172)
11. [LeetCode 172: Factorial Trailing Zeroes | Python Solution | Math & Number Theory](https://www.youtube.com/watch?v=_NxbBkunCEM)
12. [Count trailing zeroes in factorial of a number](https://www.geeksforgeeks.org/dsa/count-trailing-zeroes-factorial-number/)
13. [Find the Number of Trailing Zeroes in the Factorial of a Given Number n](https://read.learnyard.com/dsa/find-the-number-of-trailing-zeroes-in-a-given-factorial-of-n/)
14. [Leetcode Basic Calculator I, II, III](https://medium.com/@vipsb93/leetcode-basic-calculator-i-ii-iii-44362aefa752)
15. [How to Use eval() in Python](https://www.leanware.co/insights/how-to-use-eval-in-python)
16. [Basic Calculator](https://codegolf.stackexchange.com/questions/568/basic-calculator)
17. [224. Basic Calculator](https://algo.monster/liteproblems/224)
