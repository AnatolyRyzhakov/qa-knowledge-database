## Ковариантность и Контравариантность (Liskov Substitution Principle)

**RU:**
За этими сложными терминами скрывается простое правило из SOLID (Принцип подстановки Лисков - LSP). Суть принципа: **если вы используете класс-наследник вместо базового класса, код не должен сломаться.**

Чтобы это работало при переопределении методов, типизация подчиняется двум строгим правилам:
1. **Ковариантность (Covariance) для возвращаемых значений:** наследник может возвращать *более узкий* (конкретный) тип.
2. **Контравариантность (Contravariance) для аргументов:** наследник может принимать *более широкий* (абстрактный) тип.

**EN:**
Behind these complex terms lies a simple SOLID rule (Liskov Substitution Principle - LSP). The core idea: **if you use a subclass instead of a base class, the code shouldn't break.**

For this to work when overriding methods, typing follows two strict rules:
1. **Covariance for return types:** a subclass can return a *narrower* (more specific) type.
2. **Contravariance for arguments:** a subclass can accept a *broader* (more abstract) type.

---

### 1. Возвращаемые значения (Ковариантность) / Return Types (Covariance)

**RU:**
Мы имеем право **сузить** тип возвращаемого значения. Если базовый класс обещал вернуть "Транспорт", наследник может вернуть "Автомобиль". Любой код, ожидающий "Транспорт", без проблем сможет работать с "Автомобилем".

**EN:**
We are allowed to **narrow** the return type. If the base class promised to return a "Vehicle", the subclass can return a "Car". Any code expecting a "Vehicle" will have no problem working with a "Car".

#### Пример / Example:
```python
class Vehicle: pass
class Car(Vehicle): pass

class VehicleFactory:
    def create(self) -> Vehicle:
        return Vehicle()

class CarFactory(VehicleFactory):
    # УСПЕХ (Ковариантность): Мы сузили возвращаемый тип до Car.
    # SUCCESS (Covariance): We narrowed the return type to Car.
    def create(self) -> Car:
        return Car()
```
*Почему это работает?* Тот, кто вызывает `factory.create()`, ожидает получить хотя бы `Vehicle`. Получив `Car`, он не расстроится, ведь `Car` имеет все свойства `Vehicle`.

---

### 2. Входные параметры (Контравариантность) / Input Parameters (Contravariance)

**RU:**
С аргументами всё работает **наоборот**. Мы имеем право **расширить** тип принимаемого аргумента, но не сузить его! Если вы сузите тип, вы нарушите контракт (LSP).

**EN:**
With arguments, it works **the other way around**. We are allowed to **broaden** the input parameter type, but not narrow it! If you narrow the type, you violate the contract (LSP).

#### Пример (Нарушение) / Example (Violation):
```python
class Animal: pass
class Cat(Animal): pass
class SiameseCat(Cat): pass

class VetClinic:
    def treat(self, patient: Cat) -> None:
        print("Treating a cat")

class BadVetClinic(VetClinic):
    # ОШИБКА (Нарушение LSP): Мы сузили входной параметр!
    # ERROR (LSP Violation): We narrowed the input parameter!
    def treat(self, patient: SiameseCat) -> None:
        print("Treating ONLY Siamese cats")
```
*Почему это сломается?* Клиентский код уверен, что `VetClinic` может лечить **любых** котов (`Cat`). Если мы подставим `BadVetClinic` и передадим туда обычного уличного кота, программа упадет, так как эта клиника принимает только сиамских.

#### Пример (Правильно) / Example (Correct - Contravariance):
```python
class AdvancedVetClinic(VetClinic):
    # УСПЕХ (Контравариантность): Мы расширили входной параметр до Animal.
    # SUCCESS (Contravariance): We broadened the input parameter to Animal.
    def treat(self, patient: Animal) -> None:
        print("Treating any animal")
```
*Почему это работает?* Клиент ожидает, что метод примет `Cat`. Наша `AdvancedVetClinic` принимает **любых** `Animal`. Если клиент передаст туда `Cat`, клиника успешно справится с задачей. Код не сломается!

---

### 3. Применение в AutoQA / Usage in AutoQA

**RU:**
В автоматизации этот принцип постоянно встречается при реализации паттерна **Page Object**.

**EN:**
In test automation, this principle is constantly seen when implementing the **Page Object** pattern.

#### Ковариантность в Page Object (Возвращаем конкретику):
Базовая страница может возвращать абстрактную базовую страницу после клика. А конкретная страница логина после успешного входа вернет конкретную `DashboardPage`.
```python
class BasePage:
    def click_button(self) -> 'BasePage': ...

class LoginPage(BasePage):
    # Ковариантность: возвращаем более узкий класс DashboardPage
    def click_button(self) -> 'DashboardPage': ... 
```

#### Контравариантность в конфигах тестов (Принимаем обобщения):
Если базовый метод создания отчета ожидает на вход `dict`, вы можете переопределить метод так, чтобы он принимал `Mapping` (более широкий абстрактный интерфейс).
```python
from typing import Mapping

class Report:
    def generate(self, data: dict) -> None: ...

class FlexibleReport(Report):
    # Контравариантность: расширили dict до Mapping (читает любые словари)
    def generate(self, data: Mapping) -> None: ...
```

---

### 4. Резюме / Summary

Чтобы не нарушить **Liskov Substitution Principle** при наследовании:
- **Что принимаем (Аргументы):** Можно требовать *меньше* специфики (расширять тип, Контравариантность).
- **Что отдаем (Return):** Можно гарантировать *больше* специфики (сужать тип, Ковариантность).

Запомните правило: **"Требуй меньше, отдавай больше" (Be liberal in what you accept, and conservative in what you send).**
