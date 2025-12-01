## Что такое паттерн Event Bus? / What is the Event Bus Pattern?

**RU:**
**Event Bus** (Шина событий) — это паттерн, который обеспечивает связь между компонентами системы без их прямой зависимости друг от друга.

Представьте это как **общий чат** или **радиоволну**. Компонент А не звонит лично Компоненту Б. Компонент А просто пишет сообщение в "Шину". Все компоненты (Б, В, Г), которые подписаны на эту тему, получают сообщение и реагируют.

**EN:**
**Event Bus** is a pattern that enables communication between system components without them depending directly on each other.

Think of it as a **group chat** or a **radio frequency**. Component A doesn't call Component B personally. Component A simply posts a message to the "Bus". Any components (B, C, D) subscribed to that topic receive the message and react.

---

### 1. Как это работает? / How It Works?

Механизм состоит из трех частей:
1.  **Publisher (Издатель):** Отправляет событие в шину.
2.  **Event Bus (Сама шина):** Диспетчер, который хранит список подписчиков и пересылает им события.
3.  **Subscriber (Подписчик):** Говорит шине: "Если произойдет событие X, вызови меня".

---

### 2. Зачем это нужно? / Why Do We Need It?

**RU:**
Главная цель — **снижение связности (Decoupling)**.
Без шины, если вы хотите добавить отправку уведомления в Slack при падении теста, вам нужно идти в код теста и дописывать туда вызов функции Slack.
С шиной вы просто создаете нового "слушателя" (Listener), который ждет событие `TEST_FAILED`. Код теста менять не нужно.

**EN:**
The main goal is **Decoupling**.
Without a bus, if you want to add Slack notifications when a test fails, you have to go into the test code and add a function call there.
With a bus, you simply create a new "Listener" that waits for the `TEST_FAILED` event. The test code remains untouched.

---

### 3. Реализация на Python / Python Implementation

Вот простая реализация шины событий внутри одного приложения.

```python
from typing import Callable, Dict, List

class EventBus:
    def __init__(self):
        # Словарь: { "имя_события": [список_функций_обработчиков] }
        self._subscribers: Dict[str, List[Callable]] = {}

    def subscribe(self, event_type: str, handler: Callable):
        """Подписаться на событие"""
        if event_type not in self._subscribers:
            self._subscribers[event_type] = []
        self._subscribers[event_type].append(handler)

    def publish(self, event_type: str, data: dict):
        """Опубликовать событие (оповестить всех подписчиков)"""
        if event_type in self._subscribers:
            for handler in self._subscribers[event_type]:
                handler(data)

# === Использование / Usage ===

# 1. Создаем шину
bus = EventBus()

# 2. Определяем подписчиков (Listeners)
def log_failure(data):
    print(f"[LOG] Test failed: {data['test_name']}")

def send_slack_notification(data):
    print(f"[SLACK] Sending alert about {data['test_name']}...")

# 3. Подписываем их на событие "test_failed"
bus.subscribe("test_failed", log_failure)
bus.subscribe("test_failed", send_slack_notification)

# 4. Где-то в коде теста происходит ошибка (Publisher)
# Тест ничего не знает ни про логи, ни про Slack. Он знает только про Bus.
bus.publish("test_failed", {"test_name": "Login Test", "error": "Timeout"})

# Output:
# [LOG] Test failed: Login Test
# [SLACK] Sending alert about Login Test...
```

---

### 4. Event Bus в AutoQA / Event Bus in AutoQA

**RU:**
Этот паттерн повсеместно используется в тестовых фреймворках (например, в **Pytest** или **JUnit**), часто под названием **Hooks** или **Listeners**.

Когда вы пишете плагин для Pytest (например, `pytest_runtest_makereport`), вы по сути подписываетесь на события внутри шины Pytest.

**Сценарий использования:**
У вас есть автотест. Когда он падает, нужно:
1.  Сделать скриншот.
2.  Сохранить логи браузера.
3.  Отправить метрику в Grafana.

Вместо того чтобы писать эти 3 вызова в `tearDown` каждого теста, вы настраиваете Event Bus, который слушает событие падения теста и запускает все эти действия автоматически.

**EN:**
This pattern is widely used in test frameworks (like **Pytest** or **JUnit**), often referred to as **Hooks** or **Listeners**.

When you write a Pytest plugin (e.g., `pytest_runtest_makereport`), you are essentially subscribing to events inside Pytest's internal bus.

**Use Case:**
You have an automated test. When it fails, you need to:
1.  Take a screenshot.
2.  Save browser logs.
3.  Send metrics to Grafana.

Instead of writing these 3 calls in the `tearDown` of every test, you configure an Event Bus that listens for the failure event and triggers all these actions automatically.

---

### 5. Плюсы и минусы / Pros and Cons

| | RU | EN |
| :--- | :--- | :--- |
| **+** | **Легкая расширяемость.** Хотите добавить новую логику? Просто добавьте нового подписчика. Старый код трогать не надо. | **Easy Extensibility.** Want to add new logic? Just add a new subscriber. No need to touch old code. |
| **-** | **Сложная отладка.** Трудно понять, кто и в каком порядке обрабатывает событие, так как вызовы скрыты внутри шины. | **Hard Debugging.** It's difficult to trace who processes the event and in what order, as calls are hidden inside the bus. |
