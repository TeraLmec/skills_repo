### Liskov Substitution Principle

## Table of Contents

- [Liskov Substitution Principle](#liskov-substitution-principle)
- [The classic violation: Rectangle and Square](#the-classic-violation-rectangle-and-square)
- [A more realistic violation](#a-more-realistic-violation)
- [The contract rules](#the-contract-rules)


---

> **Subtypes must be substitutable for their base types.**

Barbara Liskov's formulation is more precise: if `S` is a subtype of `T`, then objects of type `T` may be
replaced with objects of type `S` without altering any of the desirable properties of the program.

In practice this means inheritance is a promise about **behaviour**, not just about shared code. A caller
that holds a reference to the base class must be able to use any subclass without knowing which one it has,
and without being surprised.

#### The classic violation: Rectangle and Square

Mathematically, a square *is a* rectangle. In code, it is not.

**Bad** :angry:

```python
class Rectangle:
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height

    def set_width(self, width: float) -> None:
        self.width = width

    def set_height(self, height: float) -> None:
        self.height = height

    def area(self) -> float:
        return self.width * self.height


class Square(Rectangle):
    def set_width(self, width: float) -> None:
        self.width = width
        self.height = width          # a square must stay square

    def set_height(self, height: float) -> None:
        self.width = height
        self.height = height
```

Now write a function against the base class:

```python
def stretch(rectangle: Rectangle) -> None:
    rectangle.set_width(5)
    rectangle.set_height(4)
    assert rectangle.area() == 20     # obviously true for a Rectangle
```

`stretch(Square(2, 2))` fails: the area is 16. The function is correct, the subclass is "correct" in
isolation, and yet together they are broken. `Square` is not substitutable for `Rectangle` because it
strengthened an invariant that callers of `Rectangle` were entitled to rely on.

**Good** :smiley: — model the shared abstraction, not the resemblance

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass


class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...


@dataclass(frozen=True)
class Rectangle(Shape):
    width: float
    height: float

    def area(self) -> float:
        return self.width * self.height


@dataclass(frozen=True)
class Square(Shape):
    side: float

    def area(self) -> float:
        return self.side ** 2
```

Notice that immutability removed the problem entirely: without setters there is no invariant for a subclass
to break. *"Prefer immutability"* is often the cheapest way to satisfy the LSP.

#### A more realistic violation

**Bad** :angry:

```python
class Account:
    def withdraw(self, amount: float) -> None:
        self.balance -= amount


class FixedDepositAccount(Account):
    def withdraw(self, amount: float) -> None:
        raise NotImplementedError('You cannot withdraw from a fixed deposit')
```

Every function that takes an `Account` and calls `withdraw` now has to know which subclass it received —
which is precisely what the base class was supposed to spare it from. Look for these smells:

- A subclass that **raises `NotImplementedError`** for an inherited method.
- A subclass whose override is **empty** ("this one does nothing").
- Callers that use `isinstance` or `type()` to decide what to do.

**Good** :smiley: — split the hierarchy along what things can actually do

```python
class Account(ABC):
    @abstractmethod
    def deposit(self, amount: Decimal) -> None: ...

    @abstractmethod
    def balance(self) -> Decimal: ...


class WithdrawableAccount(Account, ABC):
    @abstractmethod
    def withdraw(self, amount: Decimal) -> None: ...


class SavingsAccount(WithdrawableAccount): ...
class CurrentAccount(WithdrawableAccount): ...
class FixedDepositAccount(Account): ...        # simply has no withdraw()
```

Now a function that needs to withdraw asks for a `WithdrawableAccount`, and the type checker rejects a fixed
deposit at compile time instead of at 3 a.m. in production.

#### The contract rules

A subclass may not:

- **Strengthen preconditions.** If the base accepts any positive amount, the subclass may not demand amounts
  under 1000.
- **Weaken postconditions.** If the base guarantees the balance is updated, the subclass must update it.
- **Break invariants** of the base class (the `Square` case).
- **Throw new exception types** that callers of the base were not told about.

It **may** weaken preconditions (accept more) and strengthen postconditions (guarantee more).

> **Rule of thumb:** if you have to read the documentation of the subclass in order to use the base class
> safely, the LSP is broken. And if a subclass only wants *some* of the parent's behaviour, prefer
> composition over inheritance — Python's duck typing and `Protocol` classes let you share an interface
> without sharing an implementation.

**[⬆ back to top](#table-of-contents)**

