# Урок 2. ООП на практике

Сегодня пишем программу для небольшой ремонтной фирмы, которая составляет **сметы**. Смета — это список позиций: материалы, работы, прочие расходы. У каждой позиции есть количество, цена за единицу и итоговая стоимость.

На этой задаче научимся делать классы короче с помощью `@dataclass`, проверять данные при создании объекта, выстраивать иерархию «позиция → материал, работа» и соберём смету с фильтрами и сортировкой.

К концу урока у вас будет файл `estimate.py`. На следующем занятии мы разобьём его на модули и научим сохранять смету в файл.

## План урока

| Этап | Время | Что делаем |
| --- | --- | --- |
| Повторение | 0–20 мин | Добираем темы, которые на диагностике отметили как «стоит повторить» |
| Теория | 25 мин | Классы, `@dataclass`, `@property`, наследование, сортировка по ключу |
| «Что выведет?» | 10 мин | Проверяем, как поняли теорию |
| Практика | 40 мин | Пишем `estimate.py`: задачи П1–П4 |
| Итоги | 5 мин | Подводим итоги и разбираем домашнее задание |

## Подготовка

Создайте папку `lesson-02` и скачайте в неё [data.py](../files/lesson-02/data.py){ download="data.py" }. В нём смета ремонта квартиры: материалы с поставщиком, работы с размером бригады и одна прочая позиция.

```python title="data.py"
items = [
    {"id": 1, "kind": "материал", "name": "Штукатурка гипсовая, 30 кг", "quantity": 40, "unit": "меш", "unit_price": 460, "supplier": "СтройДвор"},
    {"id": 2, "kind": "работа", "name": "Штукатурка стен", "quantity": 120, "unit": "м²", "unit_price": 450, "workers": 2},
    {"id": 3, "kind": "материал", "name": "Краска интерьерная, 10 л", "quantity": 6, "unit": "шт", "unit_price": 2_900, "supplier": "Колор"},
    {"id": 4, "kind": "работа", "name": "Покраска стен", "quantity": 120, "unit": "м²", "unit_price": 250, "workers": 1},
    {"id": 5, "kind": "материал", "name": "Ламинат 33 класс", "quantity": 45, "unit": "м²", "unit_price": 1_100, "supplier": "ПолМастер"},
    {"id": 6, "kind": "работа", "name": "Укладка ламината", "quantity": 45, "unit": "м²", "unit_price": 400, "workers": 2},
    {"id": 7, "kind": "прочее", "name": "Вывоз мусора", "quantity": 2, "unit": "рейс", "unit_price": 3_500},
]
```

Поле называется `kind`, а не `type`: `type` — встроенная функция Python, и называть так свои переменные и поля не стоит.

Создайте рядом файл `estimate.py` и начните его так:

```python title="estimate.py"
from dataclasses import dataclass, field

from data import items


def format_price(n):
    return f"{n:,}".replace(",", " ") + " ₽"
```

`format_price` — это функция из задачи A1 диагностики. Можете взять своё решение.

## Теория

### Зачем классы, если есть словари

На диагностике данные хранились в словарях. Это удобно, пока программа маленькая. Но у словарей есть слабые места:

```python
item = {"id": 1, "name": "Штукатурка", "quantity": 40, "unit_price": 460}

print(item["unit_prise"])                     # опечатка: KeyError только при запуске
item["quantity"] = -5                         # никто не помешает записать бессмыслицу
total = item["quantity"] * item["unit_price"] # формулу придётся повторять везде
```

Класс собирает в одном месте данные и действия с ними. Редактор подсказывает названия полей и замечает опечатки. Проверку данных можно встроить прямо в создание объекта.

### Класс, объект, `self`

```python
class EstimateItem:
    def __init__(self, id, name, quantity, unit_price):
        self.id = id
        self.name = name
        self.quantity = quantity
        self.unit_price = unit_price

    def total(self):
        return self.quantity * self.unit_price


item = EstimateItem(1, "Штукатурка", 40, 460)   # создали объект: Python вызвал __init__
print(item.quantity)                            # 40
print(item.total())                             # 18400
```

- **Класс** — это чертёж, **объект** (экземпляр) — изделие по этому чертежу.
- `__init__` заполняет поля нового объекта.
- `self` — это сам объект. Когда вы пишете `item.total()`, Python на самом деле вызывает `EstimateItem.total(item)`.

### `__str__` и `__repr__`

Методы с двумя подчёркиваниями с каждой стороны Python вызывает сам в особых ситуациях:

- `__str__` отвечает за вид объекта в `print(obj)` и `str(obj)`. Это текст для человека.
- `__repr__` показывает объект в консоли, в отладчике и **внутри списков**. Это текст для программиста.

Если определить только `__str__`, то `print(item)` будет выглядеть красиво, а `print([item])` — нет. Проверьте это в задаче 2.1 ниже.

### `@dataclass`: класс без лишнего кода

Большая часть классов вроде `EstimateItem` просто хранит данные. Для них в Python есть декоратор `@dataclass`:

```python
from dataclasses import dataclass


@dataclass
class EstimateItem:
    id: int
    kind: str
    name: str
    quantity: float
    unit: str
    unit_price: int
```

Этих семи строк достаточно. Python сам напишет:

- `__init__`, который принимает поля в указанном порядке;
- `__repr__`, например `EstimateItem(id=1, kind='материал', ...)`;
- `__eq__`: два объекта с одинаковыми полями считаются равными.

Аннотации типов `int`, `str` здесь обязательны: по ним dataclass понимает, какие поля есть у класса. Python не проверяет, что тип значения совпадает с аннотацией. Это подсказка для людей и редактора.

**Изменяемые значения по умолчанию.** Список нельзя указать значением по умолчанию напрямую: `items: list = []` вызовет ошибку. Иначе все объекты делили бы один список, как в задаче 1.1 диагностики. Правильно так:

```python
from dataclasses import dataclass, field


@dataclass
class Estimate:
    items: list = field(default_factory=list)   # у каждой сметы свой новый список
```

### Проверка данных: `__post_init__`

Dataclass пишет `__init__` сам, поэтому свою проверку добавляют в метод `__post_init__`. Python вызывает его сразу после заполнения полей:

```python
@dataclass
class EstimateItem:
    ...

    def __post_init__(self):
        if self.quantity <= 0:
            raise ValueError(f"Количество должно быть больше нуля, получено: {self.quantity}")
```

`raise` останавливает создание объекта и сообщает об ошибке. Лучше сразу упасть с понятным сообщением, чем выставить клиенту смету с отрицательной суммой.

### `@property`: вычисляемое поле

Стоимость позиции не хранится, а вычисляется из количества и цены. Декоратор `@property` позволяет обращаться к методу как к полю, без скобок:

```python
@dataclass
class EstimateItem:
    ...

    @property
    def total(self):
        return round(self.quantity * self.unit_price)


item = EstimateItem(1, "материал", "Штукатурка гипсовая, 30 кг", 40, "меш", 460)
print(item.total)    # 18400, без скобок
```

Значение всегда актуальное: если поставщик поднял цену и вы поменяли `unit_price`, стоимость пересчитается сама.

### Наследование и `super()`

У материала есть поставщик, у работы — размер бригады. Общие поля и методы остаются в `EstimateItem`, а особенности выносятся в наследников:

```python
@dataclass
class Material(EstimateItem):
    supplier: str

    def __str__(self):
        return super().__str__() + f", поставщик {self.supplier}"
```

- `Material(EstimateItem)` означает, что `Material` получает все поля и методы `EstimateItem`.
- Новое поле `supplier` добавляется после полей родителя.
- `super().__str__()` вызывает версию метода из родителя, к которой мы дописываем своё.
- `isinstance(material, EstimateItem)` вернёт `True`: материал — это тоже позиция сметы.

### Распаковка словаря: `**`

Две звёздочки перед словарём превращают его в именованные аргументы:

```python
d = {"id": 1, "kind": "материал", "name": "Штукатурка гипсовая, 30 кг", "quantity": 40, "unit": "меш", "unit_price": 460, "supplier": "СтройДвор"}

Material(**d)
# то же самое, что:
Material(id=1, kind="материал", name="Штукатурка гипсовая, 30 кг", quantity=40, unit="меш", unit_price=460, supplier="СтройДвор")
```

Ключи словаря должны совпадать с названиями полей. Лишний или недостающий ключ вызовет `TypeError`.

### Сортировка по ключу

`sorted` умеет сортировать любые объекты, если сказать ему, по какому значению сравнивать:

```python
sorted(items, key=lambda item: item.total)                 # от дешёвых позиций к дорогим
sorted(items, key=lambda item: item.total, reverse=True)   # от дорогих к дешёвым
```

`lambda item: item.total` — это короткая функция без имени. Она получает объект и возвращает значение, по которому его сортировать. То же самое можно записать обычной функцией:

```python
def by_total(item):
    return item.total

sorted(items, key=by_total)
```

## «Что выведет?»

Сначала запишите ответ, потом запустите и сравните.

**2.1. `__str__` в списке**

```python
class Lead:
    def __init__(self, name):
        self.name = name

    def __str__(self):
        return f"Заявка: {self.name}"


leads = [Lead("Иван"), Lead("Мария")]
print(leads[0])
print(leads)
```

**2.2. Сравнение dataclass**

```python
from dataclasses import dataclass


@dataclass
class Lead:
    name: str
    phone: str


a = Lead("Иван", "+79181234567")
b = Lead("Иван", "+79181234567")
print(a == b, a is b)
print(a)
```

**2.3. Общая корзина**

```python
class Cart:
    items = []

    def add(self, x):
        self.items.append(x)


a = Cart()
b = Cart()
a.add("цемент")
print(b.items)
```

**2.4. Переопределение метода**

```python
class Base:
    def hello(self):
        return "Base"


class Child(Base):
    def hello(self):
        return "Child+" + super().hello()


print([obj.hello() for obj in [Base(), Child()]])
```

**2.5. Свойство со скобками**

```python
class Order:
    def __init__(self, price, qty):
        self.price = price
        self.qty = qty

    @property
    def total(self):
        return self.price * self.qty


o = Order(1500, 4)
print(o.total)
print(o.total())
```

## Практика

Все задачи пишутся в одном файле `estimate.py`, каждая следующая опирается на предыдущую. Проверки из блоков «Проверка» добавляйте в конец файла.

### П1. Позиция сметы

Напишите dataclass `EstimateItem` с полями `id`, `kind`, `name`, `quantity`, `unit`, `unit_price`. Добавьте:

1. Проверку в `__post_init__`: если количество меньше или равно нулю либо цена отрицательная, выбрасывается `ValueError` с понятным сообщением.
2. Свойство `total`: количество, умноженное на цену за единицу и округлённое до целого.
3. Метод `__str__` в таком формате:

```text
№1 Штукатурка гипсовая, 30 кг: 40 меш × 460 ₽ = 18 400 ₽
```

```python title="Проверка"
item = EstimateItem(1, "материал", "Штукатурка гипсовая, 30 кг", 40, "меш", 460)
assert item.total == 18400
assert str(item) == "№1 Штукатурка гипсовая, 30 кг: 40 меш × 460 ₽ = 18 400 ₽"
assert item == EstimateItem(1, "материал", "Штукатурка гипсовая, 30 кг", 40, "меш", 460)

for quantity, price in [(0, 100), (-2, 100), (5, -1)]:
    try:
        EstimateItem(1, "материал", "Тест", quantity, "шт", price)
    except ValueError:
        pass
    else:
        raise AssertionError(f"Нет ValueError для количества {quantity} и цены {price}")
```

### П2. Материалы, работы и фабрика

1. Создайте наследника `Material` с полем `supplier`. Его `__str__` дописывает в конец `, поставщик НАЗВАНИЕ`.
2. Создайте наследника `Labor` с полем `workers` (сколько человек в бригаде). Его `__str__` дописывает в конец `, бригада N чел.`.
3. Напишите функцию `from_dict(d)`, которая по словарю из `data.py` создаёт объект нужного класса: для материала `Material`, для работы `Labor`, для остального `EstimateItem`.

```python title="Проверка"
m = from_dict(items[0])
w = from_dict(items[1])
other = from_dict(items[6])

assert isinstance(m, Material) and isinstance(m, EstimateItem)
assert isinstance(w, Labor)
assert isinstance(other, EstimateItem) and not isinstance(other, (Material, Labor))

assert str(m) == "№1 Штукатурка гипсовая, 30 кг: 40 меш × 460 ₽ = 18 400 ₽, поставщик СтройДвор"
assert str(w) == "№2 Штукатурка стен: 120 м² × 450 ₽ = 54 000 ₽, бригада 2 чел."
```

### П3. Смета

Напишите dataclass `Estimate` с полем `items`: по умолчанию это пустой список. Методы сметы:

- `add(item)` — добавить позицию и вернуть `True`; если позиция с таким id уже есть, не добавлять и вернуть `False`;
- `get(id)` — вернуть позицию по id или `None`, если такой нет;
- `find(kind=None, min_total=None, max_total=None)` — вернуть список позиций, подходящих под все переданные условия; непереданные условия не учитываются;
- `sorted_by_total(reverse=False)` — вернуть позиции, отсортированные по стоимости;
- `total_cost()` — вернуть итоговую сумму сметы;
- `__len__()` — вернуть количество позиций, чтобы работал `len(estimate)`.

И отдельную функцию `load_estimate(dicts)`, которая создаёт смету из списка словарей.

```python title="Проверка"
def ids(items):
    return [item.id for item in items]

estimate = load_estimate(items)
assert len(estimate) == 7
assert estimate.add(from_dict(items[0])) is False
assert len(estimate) == 7
assert estimate.get(2).workers == 2
assert estimate.get(999) is None

assert ids(estimate.find(kind="материал")) == [1, 3, 5]
assert ids(estimate.find(min_total=20000)) == [2, 4, 5]
assert ids(estimate.find(kind="работа", max_total=30000)) == [4, 6]
assert ids(estimate.sorted_by_total()) == [7, 3, 6, 1, 4, 5, 2]
assert ids(estimate.sorted_by_total(reverse=True))[0] == 2
assert estimate.total_cost() == 194300

e1, e2 = Estimate(), Estimate()
e1.add(from_dict(items[0]))
assert len(e2) == 0, "У каждой сметы должен быть свой список"
```

!!! question "Подумайте"
    `find` очень похож на `filter_products` из задачи B2 диагностики. Что изменилось, когда фильтр стал методом сметы?

### П4. Крупнейшие статьи расходов

Клиент увидел итоговую сумму и просит показать, на что уходит больше всего денег. Напишите функцию `top_expenses(estimate, n=3, kind=None)`. Она возвращает `n` самых дорогих позиций сметы. Если передан `kind`, выбирает только среди позиций этого вида. Выведите результат через `print`.

```python title="Проверка"
assert ids(top_expenses(estimate)) == [2, 5, 4]
assert ids(top_expenses(estimate, 2, "материал")) == [5, 1]
assert top_expenses(estimate, kind="бетон") == []
```

## Бонус: игра «Что дороже?»

Программа 5 раундов подряд показывает две случайные позиции сметы: название, количество и цену за единицу. Игрок прикидывает в уме, какая позиция обойдётся дороже. После ответа программа показывает стоимость обеих позиций, а в конце — счёт.

```text
Раунд 1. Какая позиция обойдётся дороже?
1) Краска интерьерная, 10 л: 6 шт по 2 900 ₽
2) Укладка ламината: 45 м² по 400 ₽
Ваш ответ (1 или 2): 2
Верно!
Итого: 1) 17 400 ₽, 2) 18 000 ₽
```

Подсказка: два разных случайных элемента списка возвращает `random.sample(список, 2)`.

## Итоги урока

После урока вы умеете:

- [ ] объяснить, чем класс удобнее словаря для хранения данных;
- [ ] описать класс через `@dataclass` и проверить данные в `__post_init__`;
- [ ] сделать вычисляемое поле через `@property`;
- [ ] создать наследника, дописать ему поле и переопределить метод через `super()`;
- [ ] создать объект из словаря через `**`;
- [ ] отсортировать объекты по любому значению с помощью `key`.

## Домашнее задание

**1. Аренда техники.** Добавьте наследника `Equipment` для позиций вида `"техника"` с полем `days` (на сколько дней арендуем). Стоимость у техники считается иначе: количество × цена за день × число дней. Переопределите в `Equipment` свойство `total` и метод `__str__`. Научите `from_dict` создавать `Equipment`.

```python title="Проверка"
rent = from_dict({"id": 8, "kind": "техника", "name": "Аренда строительных лесов",
                  "quantity": 1, "unit": "компл", "unit_price": 900, "days": 10})
assert isinstance(rent, Equipment)
assert rent.total == 9000
assert str(rent) == "№8 Аренда строительных лесов: 1 компл × 900 ₽ × 10 дн. = 9 000 ₽"

estimate = load_estimate(items)
estimate.add(rent)
assert estimate.total_cost() == 203300
assert ids(estimate.sorted_by_total())[:2] == [7, 8]
```

Обратите внимание: методы `Estimate` вы не меняли, но они сразу правильно работают с новым видом позиций. Подумайте, почему.

**2. Обратно в словарь.** Добавьте в `EstimateItem` метод `to_dict()`, который возвращает словарь с полями объекта. Подсказка: посмотрите в документации функцию `dataclasses.asdict`. Этот метод понадобится на следующем уроке, когда будем сохранять смету в JSON.

```python title="Проверка"
estimate = load_estimate(items)
assert estimate.get(1).to_dict() == items[0]
assert all(from_dict(item.to_dict()) == item for item in estimate.items)
```

**3. Сводка по видам.** Добавьте в `Estimate` метод `stats()`, который возвращает по каждому виду позиций их количество и общую сумму.

```python title="Проверка"
assert estimate.stats() == {
    "материал": {"count": 3, "total": 85300},
    "работа": {"count": 3, "total": 102000},
    "прочее": {"count": 1, "total": 7000},
}
```

**4. Игра «Что дороже?»**, если не успели на уроке.
