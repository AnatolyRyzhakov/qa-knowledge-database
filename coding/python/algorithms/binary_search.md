## Бинарный поиск / Binary Search

**RU:**
Бинарный поиск — это эффективный алгоритм поиска элемента в **отсортированном** массиве.
Принцип работы: мы делим массив пополам. Если искомое число меньше элемента в середине, мы отбрасываем правую половину и ищем в левой. И так до тех пор, пока не найдем элемент.

**EN:**
Binary search is an efficient algorithm for finding an item in a **sorted** array.
How it works: we divide the array in half. If the target value is less than the middle element, we discard the right half and search in the left. We repeat this until the element is found.

### 1. Почему это важно? (Сравнение сложности) / Why is it important? (Complexity Comparison)

Чтобы понять мощь алгоритма, нужно сравнить его с обычным перебором.

**RU:**
Представьте, что мы ищем число в массиве из 1,000,000 элементов.
*   **Линейный поиск (Linear Search):** В худшем случае придется проверить **1,000,000** элементов $O(n)$.
*   **Бинарный поиск (Binary Search):** В худшем случае потребуется всего **20** проверок $O(\log n)$, так как $2^{20} > 1,000,000$.

**EN:**
Imagine searching for a number in an array of 1,000,000 items.
*   **Linear Search:** In the worst case, you have to check **1,000,000** items $O(n)$.
*   **Binary Search:** In the worst case, it takes only **20** checks $O(\log n)$, since $2^{20} > 1,000,000$.

---

### 2. Реализации / Implementations

#### 2.1. Итеративный подход (Классический) / Iterative Approach (Classic)

Это самый эффективный по памяти способ, так как он не использует лишних вызовов функций.

**Сложность / Complexity:**
*   Time: $O(\log n)$
*   Space: $O(1)$ (память не расходуется дополнительно)

```python
from typing import List, Optional

def binary_search_iterative(arr: List[int], target: int) -> Optional[int]:
    low = 0
    high = len(arr) - 1

    while low <= high:
        mid = (low + high) // 2
        guess = arr[mid]

        if guess == target:
            return mid  # Элемент найден / Item found
        if guess > target:
            high = mid - 1  # Ищем в левой части / Search in left half
        else:
            low = mid + 1   # Ищем в правой части / Search in right half

    return None  # Элемент не найден / Item not found

# Пример / Example:
data = [1, 3, 5, 7, 9, 11]
print(binary_search_iterative(data, 7))  # Output: 3 (index)
```

#### 2.2. Рекурсивный подход / Recursive Approach

Более "академический" стиль. Он делает то же самое, но функция вызывает сама себя.

**Сложность / Complexity:**
*   Time: $O(\log n)$
*   Space: $O(\log n)$ — **Важно:** каждый рекурсивный вызов занимает память в стеке (Stack Memory). В Python есть лимит на глубину рекурсии, поэтому итеративный метод предпочтительнее для продакшна.

```python
def binary_search_recursive(arr: List[int], target: int, low: int, high: int) -> Optional[int]:
    if low > high:
        return None

    mid = (low + high) // 2
    guess = arr[mid]

    if guess == target:
        return mid
    elif guess > target:
        return binary_search_recursive(arr, target, low, mid - 1)
    else:
        return binary_search_recursive(arr, target, mid + 1, high)

# Пример / Example:
data = [1, 3, 5, 7, 9, 11]
print(binary_search_recursive(data, 9, 0, len(data) - 1)) # Output: 4
```

---

### 3. "Pythonic way" — Модуль `bisect`

В Python есть встроенный модуль `bisect`, написанный на C. Он работает быстрее, чем любой код на чистом Python, который вы напишете сами.

**RU:** Модуль `bisect` не возвращает индекс, если элемент найден, он возвращает место, куда элемент *можно вставить*, чтобы сохранить сортировку. Но его легко адаптировать для поиска.
**EN:** The `bisect` module doesn't strictly return the index if found; it returns the insertion point to maintain order. However, it is easily adapted for searching.

```python
import bisect

def binary_search_builtin(arr, target):
    # bisect_left возвращает индекс первого вхождения элемента
    index = bisect.bisect_left(arr, target)
    
    # Проверяем, что индекс в пределах массива и элемент действительно равен искомому
    if index < len(arr) and arr[index] == target:
        return index
    return None

data = [1, 3, 5, 7, 9, 11]
print(binary_search_builtin(data, 5)) # Output: 2
```

---

### 4. Где это используется в реальной жизни? / Real-World Use Cases

Вам редко придется писать бинарный поиск с нуля, но он работает внутри многих инструментов, которые вы используете.

#### 1. Python `bisect` module
Как показано выше, стандартная библиотека Python использует этот алгоритм для вставки элементов в отсортированные списки без нарушения порядка.

#### 2. Базы данных (SQL Indices & B-Trees)
Когда вы создаете индекс в базе данных (например, `CREATE INDEX ON users(id)`), база данных строит структуру (чаще всего B-Tree), которая использует логику бинарного поиска для нахождения строк.
*   Поиск по ID без индекса: $O(n)$ (Full Table Scan).
*   Поиск по ID с индексом: $O(\log n)$ (Index Seek).

#### 3. NumPy (`np.searchsorted`)
В Data Science библиотеке NumPy функция `searchsorted` использует бинарный поиск для работы с огромными массивами данных, где скорость критически важна.

```python
import numpy as np

arr = np.array([1, 2, 3, 4, 5])
# Очень быстро находит позицию вставки / Finds insertion point extremely fast
index = np.searchsorted(arr, 3) 
```

#### 4. Git (`git bisect`)
В Git есть команда для отладки. Если вы не знаете, какой коммит сломал билд, `git bisect` позволяет вам помечать коммиты как "Good" и "Bad". Git будет делить историю коммитов пополам, пока вы не найдете виновника за $O(\log n)$ шагов вместо перебора всей истории.
