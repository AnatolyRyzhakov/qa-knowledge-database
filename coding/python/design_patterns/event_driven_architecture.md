## Что такое Event-Driven Architecture (EDA)? / What is Event-Driven Architecture (EDA)?

**RU:**
**Event-Driven Architecture (EDA)** — это архитектурный стиль, в котором компоненты системы общаются друг с другом с помощью **событий** (events).

В отличие от классического подхода "Запрос-Ответ" (Request-Response), где Сервис А прямо приказывает Сервису Б что-то сделать, в EDA Сервис А просто "кричит" в пустоту: *"Произошло событие X!"*. Сервисы, которым это интересно, слушают и реагируют.

**EN:**
**Event-Driven Architecture (EDA)** is an architectural style where system components communicate with each other using **events**.

Unlike the classic "Request-Response" approach, where Service A directly commands Service B to do something, in EDA, Service A simply "shouts" into the void: *"Event X happened!"*. Services that are interested listen and react.

---

### 1. Ключевые компоненты / Key Components

1.  **Producer (Издатель):** Тот, кто создает событие (например, "Пользователь нажал кнопку Купить").
2.  **Event (Событие):** Запись о том, что случилось (обычно JSON-объект с данными).
3.  **Broker (Брокер/Шина):** Посредник, который принимает события и раздает их подписчикам (RabbitMQ, Kafka, Redis, AWS SNS/SQS).
4.  **Consumer (Подписчик):** Тот, кто ждет события и реагирует на него (отправляет email, обновляет склад).

---

### 2. Сравнение: Ресторан / Analogy: The Restaurant

**Синхронно (Request-Response / Monolith):**
Официант принял заказ, пошел на кухню и **стоит ждет**, пока повар приготовит еду, чтобы сразу отнести её. Он не может обслуживать других, пока ждет.

**Асинхронно (Event-Driven):**
Официант вешает заказ на гвоздик ("событие создано") и уходит обслуживать других. Повар видит заказ, готовит. Когда готово, он жмет на звонок ("событие завершено"). Официант (или другой свободный сотрудник) забирает еду.

---

### 3. Зачем это нужно? / Why Do We Need It?

**RU:**
1.  **Слабая связность (Decoupling):** Сервис регистрации не знает о существовании сервиса email-рассылок. Если сервис email упадет, регистрация все равно пройдет успешно (письмо отправится позже).
2.  **Масштабируемость (Scalability):** Если пришло 1000 заказов в секунду, брокер сохранит их в очередь, и обработчики разберут их постепенно, не обрушив систему.
3.  **Асинхронность:** Пользователь не ждет, пока сгенерируется тяжелый отчет. Он получает ответ "Мы начали", а отчет придет позже.

**EN:**
1.  **Decoupling:** The registration service doesn't know the email service exists. If the email service crashes, registration still succeeds (the email will be sent later).
2.  **Scalability:** If 1000 orders arrive per second, the broker queues them, and consumers process them gradually without crashing the system.
3.  **Asynchronous processing:** The user doesn't wait for a heavy report to generate. They get a "We started" response, and the report arrives later.

---

### 4. Реализация на Python (Псевдокод) / Python Implementation (Pseudo-code)

В реальности используются библиотеки `pika` (RabbitMQ) или `kafka-python`. Но логика выглядит так:

#### Producer (Сервис заказов)
```python
# Мы не вызываем email_service.send_email() напрямую!
# Мы просто публикуем событие.

event = {
    "type": "ORDER_CREATED",
    "user_id": 123,
    "email": "user@example.com",
    "items": ["book", "pen"]
}

# Отправляем в брокер (например, RabbitMQ)
message_broker.publish("orders_queue", event)

print("Заказ принят, пользователь может идти дальше.")
```

#### Consumer (Сервис уведомлений)
```python
def callback(event):
    if event["type"] == "ORDER_CREATED":
        send_email(event["email"], "Спасибо за заказ!")
        print("Письмо отправлено.")

# Подписываемся и ждем событий вечно
message_broker.consume("orders_queue", on_message=callback)
```

---

### 5. EDA в контексте AutoQA / EDA in AutoQA Context

**RU:**
Тестирование событийно-ориентированных систем сложнее, чем обычных REST API.
**Проблема:** Вы отправляете POST-запрос на создание заказа, получаете `200 OK`, но письмо еще не отправлено (оно в очереди).
**Решение:** Нельзя делать `assert` сразу. Нужно использовать паттерн **Polling (Опрос)**.

**EN:**
Testing event-driven systems is harder than standard REST APIs.
**Problem:** You send a POST request to create an order, get `200 OK`, but the email hasn't been sent yet (it's in the queue).
**Solution:** You cannot `assert` immediately. You must use the **Polling** pattern.

```python
import time

def test_order_email_notification():
    # 1. Trigger action
    api.create_order(user_id=123)

    # 2. Wait for the asynchronous side effect (Polling)
    email_found = False
    for attempt in range(10):  # Пытаемся 10 раз
        emails = mail_server.get_emails(user_id=123)
        if emails:
            email_found = True
            break
        time.sleep(1)  # Ждем секунду перед следующей проверкой

    assert email_found, "Email was not sent within 10 seconds"
