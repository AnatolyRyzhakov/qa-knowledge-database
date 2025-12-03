## Сортировка слиянием / Merge Sort

**RU:**
Сортировка слиянием — это алгоритм, использующий принцип **"Разделяй и властвуй"** (Divide and Conquer).
Идея проста: сложную задачу (отсортировать огромный массив) мы разбиваем на множество мелких задач (отсортировать маленькие массивы), а затем "сливаем" результаты обратно в правильном порядке.

Это **устойчивая** (stable) сортировка: она сохраняет порядок одинаковых элементов, что критически важно в тестировании и работе с базами данных.

**EN:**
Merge Sort is an algorithm based on the **"Divide and Conquer"** principle.
The idea is simple: break a complex problem (sorting a huge array) into many small problems (sorting small arrays), and then "merge" the results back together in the correct order.

It is a **stable** sort: it preserves the relative order of equal elements, which is critical in testing and database operations.

---

### 1. Как это работает? / How It Works?

Алгоритм состоит из трех шагов:
1.  **Разделение (Divide):** Делим список пополам, пока не останутся кусочки длиной в 1 элемент (они считаются отсортированными).
2.  **Сортировка (Conquer):** Фактически происходит автоматически на этапе слияния.
3.  **Слияние (Merge):** Соединяем (сливаем) два отсортированных списка в один новый, сравнивая элементы по очереди.

---

### 2. Реализации / Implementations

#### 2.1. Классическая рекурсивная реализация / Classic Recursive Implementation

Это "академический" вариант. Он понятен для чтения, но не идеален по памяти, так как создает много временных копий списков (срезы `arr[:mid]`).

**Сложность / Complexity:**
*   Time: $O(n \log n)$ — всегда (и в худшем, и в лучшем случае).
*   Space: $O(n)$ — требуется дополнительная память для хранения временных списков.

```python
from typing import List

def merge_sort(arr: List[int]) -> List[int]:
    # Базовый случай: если длина 0 или 1, список уже отсортирован
    if len(arr) <= 1:
        return arr

    # 1. Разделение
    mid = len(arr) // 2
    left_half = arr[:mid]   # Создается копия! (Память!)
    right_half = arr[mid:]  # Создается копия!

    # 2. Рекурсивный вызов
    left_sorted = merge_sort(left_half)
    right_sorted = merge_sort(right_half)

    # 3. Слияние
    return merge(left_sorted, right_sorted)

def merge(left: List[int], right: List[int]) -> List[int]:
    sorted_list = []
    i = j = 0

    # Сравниваем элементы и забираем меньший
    while i < len(left) and j < len(right):
        if left[i] < right[j]:
            sorted_list.append(left[i])
            i += 1
        else:
            sorted_list.append(right[j])
            j += 1

    # Дописываем остатки (если один список кончился раньше)
    sorted_list.extend(left[i:])
    sorted_list.extend(right[j:])
    
    return sorted_list
```

#### 2.2. Оптимизированная версия (Без срезов) / Optimized Version (No Slicing)

Вместо того чтобы копировать куски массива (`left_half = arr[:mid]`), мы передаем индексы (`start`, `end`). Это экономит время на выделение памяти, хотя алгоритмическая сложность Big-O остается той же.

**Преимущество:** Меньше нагрузка на Garbage Collector и меньше потребление RAM.

```python
def merge_sort_optimized(arr: List[int], start: int, end: int):
    if start >= end:
        return

    mid = (start + end) // 2
    
    # Рекурсия без создания новых списков, передаем только индексы
    merge_sort_optimized(arr, start, mid)
    merge_sort_optimized(arr, mid + 1, end)
    
    merge_in_place(arr, start, mid, end)

def merge_in_place(arr, start, mid, end):
    # Вспомогательный метод слияния
    # В Python сложно сделать "чистый" in-place merge эффективно без C-расширений,
    # поэтому здесь создается временный буфер только для сливаемой части.
    left = arr[start : mid + 1]
    right = arr[mid + 1 : end + 1]
    
    i = j = 0
    k = start # Записываем сразу в основной массив
    
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            arr[k] = left[i]
            i += 1
        else:
            arr[k] = right[j]
            j += 1
        k += 1

    while i < len(left):
        arr[k] = left[i]
        i += 1
        k += 1
    # Остатки right копировать не нужно, они и так уже на месте
```

---

### 3. Где это реализовано? (Real-World)

Вам почти никогда не нужно писать Merge Sort вручную, потому что он лежит в основе самых мощных встроенных сортировок.

#### 1. Python (`Timsort`)
Стандартная сортировка Python (`list.sort()` и `sorted()`) использует алгоритм **Timsort**.
Это гибрид:
*   Для маленьких массивов (или кусочков большого) используется **Insertion Sort** (Сортировка вставками), так как на малых данных она быстрее.
*   Затем эти кусочки сливаются с помощью **Merge Sort**.
Это делает Python-сортировку невероятно эффективной на реальных данных (особенно частично отсортированных).

#### 2. Внешняя сортировка (External Sorting)
**Сценарий:** У вас есть лог-файл на 100 ГБ, а оперативной памяти (RAM) всего 8 ГБ. Как его отсортировать?
QuickSort не сработает (надо грузить всё в память).
Здесь используют **Merge Sort**:
1.  Читаем файл кусками по 8 ГБ.
2.  Сортируем кусок в памяти и пишем во временный файл.
3.  Повторяем.
4.  В конце "сливаем" (Merge) 12 отсортированных файлов в один итоговый, читая их построчно (стриминг).

---

### 4. Резюме / Summary

| Свойство / Property | Значение / Value | Пояснение / Explanation |
| :--- | :--- | :--- |
| **Time Complexity** | $O(n \log n)$ | Гарантировано всегда. |
| **Space Complexity** | $O(n)$ | Требует доп. память (в отличие от QuickSort). |
| **Stability** | Да | Если `A == B` и `A` было раньше, оно останется раньше. |
| **Best Use Case** | Связные списки, Огромные данные (HDD) | Когда важна стабильность или данных больше, чем RAM. |
