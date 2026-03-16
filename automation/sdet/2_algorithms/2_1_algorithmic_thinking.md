# 📘 Глава 2. Алгоритмы и Live Coding (Подготовка к FAANG/BigTech)
## Тема 2.1. Алгоритмическое мышление: Оценка сложности $O(n)$ и выбор структур данных

### Введение: Зачем SDET знать алгоритмы в 2026 году?
В компаниях уровня FAANG (Meta, Amazon, Apple, Netflix, Google) роль SDET (Software Development Engineer in Test) кардинально отличается от классического тестировщика. SDET — это полноценный разработчик, который пишет код для проверки другого кода. На собеседованиях в FAANG к SDET предъявляются те же требования по знанию алгоритмов и структур данных, что и к обычным Software Engineers (SWE). 

Зачем автоматизатору алгоритмы? Когда фреймворк тестирует микросервисы, оперирующие сотнями тысяч записей, неоптимизированный код автотеста может стать "бутылочным горлышком". Понимание временной (Time) и пространственной (Space) сложности позволяет писать скрипты, которые не падают по таймауту и не исчерпывают оперативную память (OOM) при масштабировании.

---

### Часть 1. Оценка сложности: Big O Notation (Нотация "О" большое)

Big O Notation — это математический способ описать эффективность алгоритма. Он показывает, как меняется время выполнения (Time Complexity) или потребление памяти (Space Complexity) в худшем случае при росте объема входных данных. 

Для SDET особенно важно понимать следующие классы сложности:

1.  **$O(1)$ — Константное время (Constant Time)**
    Время выполнения не меняется независимо от объема данных.
    *Пример в автотестах:* Получение токена пользователя из словаря (`dict`) по известному ключу (ID пользователя). Поиск по 10 ключам и по 1 000 000 ключей займет одинаковое время.
2.  **$O(\log n)$ — Логарифмическое время (Logarithmic Time)**
    Время выполнения растет очень медленно. С каждым шагом алгоритм отбрасывает половину данных.
    *Пример в автотестах:* Использование бинарного поиска (Binary Search) для нахождения нужного `timestamp` в огромном, предварительно отсортированном лог-файле.
3.  **$O(n)$ — Линейное время (Linear Time)**
    Время выполнения растет пропорционально размеру данных.
    *Пример в автотестах:* Проход циклом `for` по списку из всех веб-элементов в таблице для проверки их текстовых значений.
4.  **$O(n \log n)$ — Линейно-логарифмическое время**
    Это стандартная сложность для самых эффективных алгоритмов сортировки (Merge Sort, Quick Sort).
    *Пример в автотестах:* Сортировка массива цен товаров, полученного через API, перед его сравнением с эталонным массивом из базы данных.
5.  **$O(n^2)$ — Квадратичное время (Квадратная сложность — Красный флаг 🚩)**
    Возникает при использовании вложенных циклов. Время растет стремительно и ведет к таймаутам.
    *Пример в автотестах:* Сравнение каждого элемента первого списка (10,000 JSON ответов) с каждым элементом второго списка. Это приведет к 100,000,000 операций.

---

### Часть 2. Выбор правильных структур данных (Data Structures)

Алгоритмическое мышление строится на правиле компромисса (Space-Time Tradeoff): часто имеет смысл потратить немного больше оперативной памяти на создание правильной структуры данных, чтобы кардинально сократить время выполнения кода. Для SDET ключевыми являются следующие структуры:

#### 1. Hash Maps (Хэш-таблицы / Словари `dict` в Python)
*   **Сложность:** Поиск, вставка и удаление выполняются в среднем за **$O(1)$**.
*   **Применение:** Это самый мощный инструмент автоматизатора. Вместо $O(n^2)$ сравнения двух массивов, один массив превращается в словарь (где ключом выступает уникальный ID записи), а второй массив проверяется через быстрый поиск по ключам этого словаря.

#### 2. Sets (Множества `set` в Python)
*   **Сложность:** Поиск наличия элемента (`if item in set`) выполняется за **$O(1)$**.
*   **Применение:** Идеально подходит для проверки уникальности данных или поиска пересечений. Если автотесту нужно проверить, что API не вернуло дубликатов среди 50,000 записей, создание множества и сравнение длин (`len(data) == len(set(data))`) решает задачу моментально.

#### 3. Arrays и Linked Lists (Массивы и Связные списки)
*   **Массивы (Списки `list`):** Хороши для хранения упорядоченных данных и доступа по индексу ( $O(1)$ ). Однако поиск элемента по значению или удаление элемента из начала массива занимает $O(n)$ времени, так как требует сдвига всех остальных элементов в памяти.
*   **Связные списки (Linked Lists):** Полезны, когда необходимо постоянно добавлять или удалять элементы из начала/середины коллекции, не сдвигая весь массив.

#### 4. Queues & Stacks (Очереди и Стеки)
*   **Queue (Очередь - FIFO):** Применяется для тестирования событийно-ориентированной архитектуры (EDA), проверки брокеров сообщений (RabbitMQ, Kafka) и организации пулов потоков.
*   **Stack (Стек - LIFO):** Используется для обхода деревьев, парсинга вложенных структур (глубокие XML или JSON) и проверки функционала "Отмена действия" (Undo) в UI.

#### 5. Trees & Graphs (Деревья и Графы)
Графы и алгоритмы их обхода (DFS, BFS) — стандартный вопрос на интервью в FAANG.
*   **Применение в SDET:** Тестирование систем связей (социальные графы, маршрутизаторы), генерация карт покрытия автотестами или навигация по сложному DOM-дереву страницы.

---

### Часть 3. Best Practices & Bad Practices (SDET Контекст)

Понимание Big O помогает писать качественный код на этапе код-ревью.

#### ❌ Bad Practices (Код, который провалит интервью)

**Поиск в списке внутри цикла (Silent $O(n^2)$ ):**
Использование оператора `in` для обычного списка (`list`) внутри цикла `for`.
```python
# ПЛОХО: O(n) для внешнего цикла * O(n) для внутреннего поиска = O(n^2)
actual_ids =[101, 102, 103, ... ] # 100,000 элементов из API
expected_ids =[102, 105, ... ] # 10,000 элементов из БД

for expected in expected_ids:
    if expected in actual_ids:  # Поиск в списке занимает линейное время O(n)!
        validate(expected)
```
*На больших объемах данных этот автотест зависнет (потребует 1 миллиард операций).*

#### ✅ Best Practices (Уровень FAANG SDET)

**Оптимизация с помощью Space-Time Tradeoff (Использование Set):**
Преобразование целевого списка в множество (Set) перед началом поиска.
```python
# ОТЛИЧНО: O(n) на конвертацию + (O(n) цикла * O(1) поиска) = O(n) линейное время
actual_ids_set = set(actual_ids) # Тратим немного памяти на создание Set

for expected in expected_ids:
    if expected in actual_ids_set:  # Поиск в множестве занимает O(1)!
        validate(expected)
```
*Количество операций снижается с 1,000,000,000 до ~110,000. Тест выполнится за миллисекунды.*

Sources:
1. [How are the data structures and algorithms useful for SDET?](https://alliancedevlabs.medium.com/how-are-the-data-structures-and-algorithms-useful-for-sdet-5d6721a99be3)
2. [Do software engineers at FAANG see and write algorithms like the ones asked in their interviews, or is it a complete different "world"?](https://www.quora.com/Do-software-engineers-at-FAANG-see-and-write-algorithms-like-the-ones-asked-in-their-interviews-or-is-it-a-complete-different-world)
3. [Data Structures and Algorithms for SDET Interview Preparation in Java](https://medium.com/@laksr14/data-structures-and-algorithms-for-sdet-interview-preparation-in-java-e6d4172dc8d8)
4. [What is Big O notation, and why is it important in analyzing algorithms?](https://community.testmuai.com/t/what-is-big-o-notation-and-why-is-it-important-in-analyzing-algorithms/27512)
5. [Time & Space Complexity (Big O Notation)](https://medium.com/@fulton_shaun/time-space-complexity-big-o-notation-6c53ff101ee6)
6. [Visualizing Big O Notation in 12 Minutes](https://medium.com/better-programming/visualizing-big-o-notation-in-12-minutes-e221fc4cd2e3)
7. [The FAANG Software Engineering Interview Process](https://www.interviewhelp.io/blog/posts/faang-software-engineer-interview-process/)
8. [How are the data structures and algorithms useful for SDET?](https://www.devlabsalliance.com/blog/how-are-the-data-structures-and-algorithms-useful-for-sdet)
9. [How to Study for Data-Structures and Algorithms Interviews at FAANG](https://medium.com/swlh/how-to-study-for-data-structures-and-algorithms-interviews-at-faang-65043e00b5df)
10. [How Data Structures and Algorithms is useful for QA Automation Testers?](https://www.syntaxtechs.com/blog/how-data-structures-and-algorithm-is-useful-for-sdets/)
