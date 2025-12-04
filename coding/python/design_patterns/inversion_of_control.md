## Что такое IoC? / What is IoC?

**RU:**
**Inversion of Control (Инверсия управления)** — это архитектурный принцип, при котором поток выполнения программы контролируется не вашим кодом, а внешним фреймворком или контейнером.

В традиционном программировании **вы** вызываете функции библиотеки.
В IoC **фреймворк** вызывает ваш код.

Этот принцип также известен как **Голливудский принцип**:
> *"Не звоните нам, мы сами вам позвоним".*

**EN:**
**Inversion of Control (IoC)** is an architectural principle where the flow of control is determined by an external framework or container, not by your custom code.

In traditional programming, **you** call the library functions.
In IoC, the **framework** calls your code.

This principle is also known as the **Hollywood Principle**:
> *"Don't call us, we'll call you."*

---

### 1. Библиотека vs Фреймворк / Library vs Framework

Лучший способ понять IoC — сравнить библиотеку и фреймворк.

#### Традиционный подход (Библиотека)
Вы — водитель. Вы решаете, куда повернуть и когда нажать на тормоз.

```python
# Вы контролируете поток / You control the flow
import requests

print("Start")
response = requests.get("https://google.com") # Вы явно вызываете функцию
if response.status_code == 200:
    print("Success")
print("End")
```

#### Подход IoC (Фреймворк / Pytest)
Вы — пассажир такси. Вы просто говорите "хочу в аэропорт" (пишете тест), а таксист (Pytest) сам решает, каким маршрутом ехать, когда переключать передачи и когда вас высадить.

```python
# Вы НЕ вызываете эту функцию сами. Pytest найдет её и вызовет.
# You DO NOT call this function yourself. Pytest will find and call it.
def test_google_search():
    assert 1 + 1 == 2
```
*Здесь Pytest управляет запуском, порядком выполнения, перехватом ошибок и отчетами. Управление инвертировано.*

---

### 2. IoC и Dependency Injection (DI)

**Важно:** IoC — это **принцип** (идея). DI (Внедрение зависимостей) — это конкретный **паттерн**, реализующий этот принцип.

**Без IoC (Жесткая связность):**
Класс сам создает свои зависимости. Он контролирует их жизненный цикл.

```python
class Database:
    def connect(self): pass

class App:
    def __init__(self):
        # App сам решает, какую БД создать. Он "главный".
        self.db = Database() 
```

**C IoC (Через DI):**
Класс пассивен. Кто-то извне (IoC-контейнер) создает зависимости и "впрыскивает" их в класс.

```python
class App:
    # App говорит: "Мне нужна какая-то БД, дайте мне её".
    def __init__(self, db): 
        self.db = db
```

---

### 3. Зачем это нужно? / Why Do We Need It?

1.  **Слабая связность (Decoupling):** Ваш код фокусируется на бизнес-логике, а не на склеивании компонентов.
2.  **Легкая заменяемость:** Хотите заменить SQL-базу на тестовую Mock-базу? В традиционном коде придется переписывать `__init__`. В IoC вы просто передаете другой объект.
3.  **Тестируемость:** Благодаря IoC мы можем легко подменять реальные сервисы на заглушки (Mocks) во время тестов.

---

### 4. IoC в AutoQA (На примере Pytest) / IoC in AutoQA

Pytest — это чистейший пример IoC.

#### Пример: Фикстуры (Fixtures)

Когда вы пишете тест, вы не создаете драйвер браузера внутри теста. Вы просто указываете аргумент `driver`.
Pytest (как IoC-контейнер):
1.  Видит, что тесту нужен `driver`.
2.  Ищет фикстуру с таким именем.
3.  Запускает её.
4.  Передает результат в ваш тест.

```python
import pytest

# 1. Мы регистрируем зависимость в контейнере Pytest
@pytest.fixture
def driver():
    print("Open Browser")
    yield "Chrome"
    print("Close Browser")

# 2. IoC в действии: мы не вызываем driver(), мы просто просим его
def test_title(driver):
    # Pytest САМ вызвал driver() и передал результат сюда
    assert driver == "Chrome"
```

Если бы не было IoC, тесты выглядели бы так (ужасно):

```python
# Без IoC (плохо)
def test_title_bad():
    d = driver() # Сами создаем
    try:
        assert d == "Chrome"
    finally:
        # Сами должны помнить про закрытие
        # Сами должны обрабатывать ошибки
        pass 
```

---

### 5. Резюме / Summary

| Концепция | Описание | Пример |
| :--- | :--- | :--- |
| **Traditional Flow** | Ваш код вызывает библиотеки. | `requests.get()`, `math.sqrt()` |
| **IoC (Inversion of Control)** | Фреймворк вызывает ваш код. | `def test_name():`, `def on_click():` |
| **Главный принцип** | "Don't call us, we'll call you". | Голливудский принцип. |
| **Связь с DI** | IoC — это "ЧТО мы хотим" (принцип). DI — это "КАК мы это делаем" (паттерн). | Передача драйвера через аргументы теста. |
