## **Objects and Data Structures**

## Table of Contents

- [Objects and Data Structures](#objects-and-data-structures)
- [The Law of Demeter](#the-law-of-demeter)
- [The data and object anti-symmetry](#the-data-and-object-anti-symmetry)
- [Data Transfer Objects](#data-transfer-objects)


---

There is a distinction hiding in most codebases that few people make explicit:

- An **object** hides its data behind abstractions and exposes behaviour that operates on that data.
- A **data structure** exposes its data and has essentially no meaningful behaviour.

Both are legitimate. What causes pain is the hybrid — a class with private attributes, a getter and a setter
for each of them, and no behaviour. That is a data structure wearing an object's coat: it pays the cost of
encapsulation without getting any of the benefit.

**Bad** :angry:

```python
class Vehicle:
    def __init__(self, fuel_tank_capacity: float, fuel_level: float) -> None:
        self._fuel_tank_capacity = fuel_tank_capacity
        self._fuel_level = fuel_level

    def get_fuel_tank_capacity(self) -> float:
        return self._fuel_tank_capacity

    def get_fuel_level(self) -> float:
        return self._fuel_level
```

```python
# every caller has to know how to compute this
percentage = vehicle.get_fuel_level() / vehicle.get_fuel_tank_capacity() * 100
```

The class hides nothing: the callers know the tank is measured in litres and that a percentage is the ratio
of the two. Change the internal representation and every call site breaks.

**Good** :smiley:

```python
class Vehicle:
    def __init__(self, fuel_tank_capacity: float, fuel_level: float) -> None:
        self._fuel_tank_capacity = fuel_tank_capacity
        self._fuel_level = fuel_level

    def percent_fuel_remaining(self) -> float:
        return self._fuel_level / self._fuel_tank_capacity * 100

    def refuel(self, litres: float) -> None:
        self._fuel_level = min(self._fuel_level + litres, self._fuel_tank_capacity)
```

The caller asks the vehicle a question in the vehicle's own terms. How fuel is stored is now genuinely
private and can change freely.

### The Law of Demeter

A method should only talk to its immediate friends: itself, its own attributes, its arguments and objects it
creates. It should not reach through one object to get at another.

**Bad** :angry: — a "train wreck"

```python
output_dir = ctx.get_options().get_scratch_dir().get_absolute_path()
```

That single line couples the caller to `ctx`, to options, to scratch directories and to paths. Any of the
four can break it.

**Good** :smiley:

```python
output_dir = ctx.scratch_directory_path()
```

Ask the object for what you want, do not navigate its internals to get it yourself. A useful phrasing:
**Tell, don't ask.**

**Bad** :angry:

```python
if order.customer.subscription.status == 'active':
    order.apply_discount(0.1)
```

**Good** :smiley:

```python
if order.customer.is_subscribed():
    order.apply_discount(0.1)
```

Note that the law applies to *objects*, not to data structures. `config['db']['host']` is not a violation:
a dict is a data structure and has no behaviour to hide.

### The data and object anti-symmetry

Objects and data structures are opposites, and each is good at exactly what the other is bad at:

- **Objects** make it easy to add new *types* without changing existing functions, and hard to add new
  *operations* (every class must implement them).
- **Data structures** make it easy to add new *operations* without changing existing types, and hard to add
  new *types* (every function must handle them).

This is why "everything should be an object" is bad advice, and so is its opposite. Choose based on which
axis you expect to grow.

```python
# Data structure + functions: easy to add new operations, painful to add a new shape
@dataclass(frozen=True)
class Rectangle:
    width: float
    height: float


@dataclass(frozen=True)
class Circle:
    radius: float


def area(shape: Rectangle | Circle) -> float:
    match shape:
        case Rectangle(width=w, height=h):
            return w * h
        case Circle(radius=r):
            return math.pi * r ** 2
```

```python
# Objects: easy to add a new shape, painful to add a new operation to all of them
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...


class Rectangle(Shape):
    def area(self) -> float: ...


class Circle(Shape):
    def area(self) -> float: ...
```

### Data Transfer Objects

The purest data structure is a class with public fields and no functions at all. Python has first class
support for them, and you should use it rather than passing raw dicts around:

```python
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)
class OrderLine:
    sku: str
    quantity: int
    unit_price: Decimal
```

Compared with `{'sku': ..., 'quantity': ..., 'unit_price': ...}`, the dataclass gives you a name for the
concept, a place to document it, autocompletion, type checking, equality and an obvious place to add
behaviour when the day comes. `frozen=True` also makes it hashable and safe to share.

For DTOs crossing an untrusted boundary (an HTTP request, a message queue), a validating library such as
`pydantic` is the natural extension of the same idea: parse once at the edge, and pass typed objects
everywhere inside.

> **Rule of thumb:** hide data behind behaviour when the data has rules; expose it plainly when it does not.
> Do not do both.

**[⬆ back to top](#table-of-contents)**

