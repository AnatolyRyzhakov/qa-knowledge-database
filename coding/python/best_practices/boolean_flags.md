## Что такое Булевы флаги? / What are Boolean Flags?

**RU:**
Булевы флаги — это переменные или атрибуты класса типа `bool` (`True` / `False`), которые хранят информацию о **состоянии** объекта или его **возможностях**.

Главное правило хорошего тона: имя флага должно звучать как **вопрос**, на который можно ответить "Да" или "Нет".

**EN:**
Boolean flags are variables or class attributes of type `bool` (`True` / `False`) that store information about an object's **state** or **capabilities**.

The golden rule: a flag's name should sound like a **question** that can be answered with "Yes" or "No".

---

### 1. Нейминг: Как называть флаги? / Naming Conventions

В Python (и в разработке в целом) приняты стандартные префиксы.

#### 1.1. `is_` (Является ли? / Находится ли в состоянии?)
Самый частый префикс. Описывает текущее состояние или идентичность.

*   `is_visible`: Элемент видим?
*   `is_admin`: Пользователь админ?
*   `is_empty`: Список пуст?
*   `_is_initialized`: Объект уже настроен? (Приватный флаг)

#### 1.2. `has_` (Имеет ли? / Содержит ли?)
Описывает наличие чего-либо (данных, прав, детей).

*   `has_errors`: Есть ли ошибки в ответе?
*   `has_permission`: Есть ли доступ?
*   `has_children`: Есть ли вложенные элементы?

#### 1.3. `can_` / `should_` (Может ли? / Должен ли?)
Описывает возможности или намерения.

*   `can_edit`: Может ли пользователь редактировать?
*   `should_retry`: Нужно ли повторить запрос?

#### Анти-паттерн: Отрицательные имена
Никогда не используйте отрицания в названиях флагов. Двойное отрицание (`if not is_not_valid`) взрывает мозг.

*   `is_not_found` (Плохо / Bad) -> `is_found` (Хорошо / Good)
*   `disable_ssl` (Плохо / Bad) -> `enable_ssl` (Хорошо / Good)

---

### 2. Типовые сценарии использования / Common Use Cases

#### 2.1. Ленивая инициализация (`_is_initialized`)
Используется, когда настройка объекта тяжелая (например, логин в систему), и мы хотим сделать её только один раз при первом обращении.

```python
class ApiClient:
    def __init__(self):
        self._is_initialized = False
        self._token = None

    def connect(self):
        if self._is_initialized:
            return  # Уже готово, выходим
        
        print("Authenticating...")
        self._token = "new_token"
        self._is_initialized = True
```

#### 2.2. Кэширование и "Грязные" данные (`is_dirty`)
Паттерн, пришедший из ORM и UI. Флаг `is_dirty` означает, что данные в объекте изменились, но еще не сохранены в базу (или не перерисованы).

```python
class UserSettings:
    def __init__(self):
        self.theme = "light"
        self._is_dirty = False  # Данные синхронизированы

    def set_theme(self, new_theme):
        self.theme = new_theme
        self._is_dirty = True   # Нужна синхронизация!

    def save(self):
        if not self._is_dirty:
            return # Нечего сохранять
        # ... отправка на сервер ...
        self._is_dirty = False
```

#### 2.3. Управление ресурсами (`is_connected`, `is_closed`)
Чтобы не пытаться читать из закрытого файла или писать в разорванное соединение.

```python
class Database:
    def close(self):
        if self.is_closed:
            return # Уже закрыто
        self.connection.close()
        self.is_closed = True
```

---

### 3. Когда флаги — это ПЛОХО? / When are Flags BAD?

#### 3.1. Flag Argument (Флаг-аргумент функции)
Это нарушение принципа единственной ответственности (Single Responsibility Principle). Если вы передаете `bool` в функцию, значит, ваша функция делает **две разные вещи**.

**Плохо / Bad:**
```python
def render(is_admin: bool):
    if is_admin:
        show_admin_panel()
    else:
        show_user_panel()

# Вызов непонятен без контекста:
render(True) 
```

**Хорошо / Good:**
Разделите на две функции.
```python
def render_admin_panel(): ...
def render_user_panel(): ...
```

#### 3.2. Взрыв состояний (State Explosion)
Если у вас в классе много флагов, которые зависят друг от друга, это знак, что пора рефакторить.

**Плохо / Bad:**
```python
class Order:
    is_created = True
    is_paid = False
    is_shipped = False
    is_delivered = False
    is_cancelled = False
```
*Проблема:* А может ли быть `is_paid = True`, но `is_created = False`? Как гарантировать, что заказ не `is_cancelled` и `is_shipped` одновременно?

**Решение: Используйте Enum (Перечисление)**
Когда состояния **взаимоисключающие**, используйте `Enum`, а не кучу булевых переменных.

```python
from enum import Enum, auto

class OrderStatus(Enum):
    CREATED = auto()
    PAID = auto()
    SHIPPED = auto()
    DELIVERED = auto()
    CANCELLED = auto()

class Order:
    status: OrderStatus = OrderStatus.CREATED
```

---

### 4. Резюме / Summary

| Ситуация | Решение |
| :--- | :--- |
| **Состояние "Да/Нет"** | Используйте `bool` (`is_visible`, `has_access`). |
| **Действие "Сделать А или Б"** | Не передавайте флаг. Сделайте две функции. |
| **Много взаимоисключающих флагов** | Не используйте `bool`. Используйте `Enum`. |
| **Название** | Всегда должно звучать как вопрос (`is_...`, `can_...`). Избегайте отрицаний. |
