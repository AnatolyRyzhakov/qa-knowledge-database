# 📘 Глава 2. Алгоритмы и Live Coding (Подготовка к FAANG/BigTech)
## Тема 2.2. Манипуляция данными: Рекурсивный парсинг (flatten) сложных JSON/словарей

### Введение: Зачем это спрашивают на интервью?
На алгоритмических интервью от вас ожидают не просто написания работающего кода, но и понимания ограничений языка Python. Интервьюер обязательно спросит: *"А что произойдет, если вложенность JSON составит 2000 уровней?"*. Здесь происходит водораздел между Middle-специалистом (который напишет классическую рекурсию) и Senior (который напишет итеративный обход с использованием стека).

---

### Часть 1. Базовый уровень: Рекурсивный подход

Рекурсия — это функция, которая вызывает саму себя. Это наиболее очевидный способ обойти дерево (JSON) в глубину.

**Задача:** Превратить вложенный JSON в плоский словарь.
```python
# Исходный JSON
unflat_json = {
    'user': {
        'Rachel': {
            'UserID': 171717,
            'Email': 'rachel@test.com'
        }
    }
}
# Ожидаемый результат: {'user_Rachel_UserID': 171717, 'user_Rachel_Email': 'rachel@test.com'}
```

**Решение на интервью (Рекурсия):**
```python
def flatten_json_recursive(nested_json, parent_key='', sep='_'):
    """Базовый рекурсивный алгоритм уплощения словаря"""
    items =[]
    for k, v in nested_json.items(): # Итерируемся по ключам и значениям словаря
        new_key = f"{parent_key}{sep}{k}" if parent_key else k
        
        if isinstance(v, dict):
            # Если значение - словарь, уходим на уровень глубже (рекурсивный вызов)
            items.extend(flatten_json_recursive(v, new_key, sep=sep).items())
        else:
            # Если базовый тип (строка, число), добавляем в результат
            items.append((new_key, v))
            
    return dict(items)
```

**Плюсы:** Код лаконичный и читаемый.

**Минусы:** Опасно для Production. Интерпретатор CPython имеет жесткое ограничение на глубину стека вызовов (обычно 1000 вызовов). Если база данных вернет JSON с аномальной вложенностью, ваш скрипт упадет с ошибкой `Traceback: maximum recursion depth exceeded`.

---

### Часть 2. Архитектурный уровень (Senior SDET): Итеративный подход

**Проблема Рекурсии:** Потенциальный Stack Overflow на глубоких структурах. Увеличивать лимит через `sys.setrecursionlimit()` в CI/CD — это антипаттерн.

**Решение FAANG:** Использовать **Итеративный подход (Iterative Approach)**. Вместо использования системного стека вызовов функций, мы создаем собственный стек (структуру данных `list`) внутри памяти. Это позволяет обрабатывать JSON абсолютно любой глубины, не боясь системного переполнения.

**Идеальное решение на интервью (Итеративный обход со стеком):**
```python
def flatten_json_iterative(nested_json, sep='_'):
    """Безопасный итеративный алгоритм уплощения"""
    out = {}
    # Стек хранит кортежи: (текущий_словарь, составной_родительский_ключ)
    stack =[(nested_json, '')] # Инициализируем стек корневым словарем
    
    while stack: # Цикл работает, пока стек не опустеет
        current_dict, parent_key = stack.pop()
        
        for k, v in current_dict.items():
            new_key = f"{parent_key}{sep}{k}" if parent_key else k
            
            if isinstance(v, dict):
                # Кладем вложенный словарь обратно в стек для последующей обработки
                stack.append((v, new_key))
            else:
                out[new_key] = v # Сохраняем скалярное значение
                
    return out
```
В данном подходе сложность по времени составляет $O(n)$, где n — количество всех узлов в JSON, так как мы посещаем каждый элемент ровно один раз. Пространственная сложность также оценивается как $O(n)$.

---

### Часть 3. Продвинутый Edge Case: Обработка массивов (Arrays/Lists)

В реальном мире JSON почти всегда содержит массивы. Обычный алгоритм оставит массив нетронутым (он не является словарем). На интервью уровня Senior ожидают, что вы обработаете массивы, превратив их индексы в часть составного ключа.

**Доработка алгоритма для поддержки списков:**
```python
            if isinstance(v, dict):
                stack.append((v, new_key))
            elif isinstance(v, list):
                # Превращаем список в словарь вида {'0': val1, '1': val2} 
                # и кладем в стек для стандартной обработки
                dict_from_list = {str(i): item for i, item in enumerate(v)}
                stack.append((dict_from_list, new_key))
            else:
                out[new_key] = v
```
Таким образом структура вида `{"friends": ["John", "Jeremy"]}` превратится в плоские ключи `friends_0: "John"` и `friends_1: "Jeremy"`.

---

### Часть 4. Best Practices & Bad Practices (SDET Контекст)

#### ❌ Bad Practices (Код, который не пройдет код-ревью)
1.  **Потеря ключей:** Неправильная обработка пустых ключей `""`. Если алгоритм склеивает ключи через точку (`.`), пустой ключ может привести к двойной точке `..` или перезаписи данных (коллизии ключей).
2.  **Изменение словаря во время итерации:** Использование конструкции `del dictionary[key]` во время цикла. Это вызовет `RuntimeError`. Всегда создавайте новый словарь (`out = {}`), а не пытайтесь "мутировать" исходный.
3.  **Использование рекурсии на неизвестных данных:** Применение рекурсивной функции для парсинга ответов от сторонних API, которые вы не контролируете. Непредсказуемая вложенность убьет ваш фреймворк.

#### ✅ Best Practices 
1.  **Итеративный подход как стандарт:** Итеративный метод (со стеком) обеспечивает более контролируемый поток выполнения и спасает от падений на огромных датасетах.
2.  **Генераторы для экономии памяти:** Если вы не хотите хранить весь плоский словарь в памяти, функцию можно превратить в генератор (`yield key, value`), отдавая пары ключ-значение "лениво".
3.  **Использование готовых библиотек в Production:**
    На интервью вы обязаны написать алгоритм сами. Но в реальном рабочем проекте писать кастомные "велосипеды" не принято. Senior SDET использует библиотеку `pandas` (`pandas.json_normalize` для датафреймов) или стандартизированные пакеты (например, `flatten_dict`).

Sources:
  1. [Flattening a JSON Object Using Recursion in Python](https://www.youtube.com/watch?v=aUuleLd0lwM)
  2. [Python - Distinct Flatten dictionaries](https://www.geeksforgeeks.org/python/python-distinct-flatten-dictionaries/)
  3. [FrontendLead - Flatten I](https://frontendlead.com/coding-questions/flatten-arrays-recursively-and-iteratively)
  4. [Flatten Dictionary Python challenge](https://codereview.stackexchange.com/questions/186013/flatten-dictionary-python-challenge)
  5. [Flatten a Dictionary](https://www.tryexponent.com/courses/swe-practice/flatten-a-dictionary)
  6. [How do you flatten a dictionary in Python?](https://www.quora.com/How-do-you-flatten-a-dictionary-in-Python)
  7. [How to flatten a nested json array?](https://stackoverflow.com/questions/68695801/how-to-flatten-a-nested-json-array)
