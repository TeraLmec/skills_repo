### Dependency Inversion Principle

## Table of Contents

- [Dependency Inversion Principle](#dependency-inversion-principle)
- [The violation](#the-violation)
- [What it buys you](#what-it-buys-you)
- [Doing it the Pythonic way](#doing-it-the-pythonic-way)


---

> **A. High-level modules should not depend on low-level modules. Both should depend on abstractions.**
> **B. Abstractions should not depend on details. Details should depend on abstractions.**

The word "inversion" refers to the direction of the source-code dependency. In a naive design, business
logic imports the database driver, so the arrow points from policy down to detail. Inverted, the business
logic defines the interface it needs and the database module implements it — the arrow now points *up*,
towards the policy.

#### The violation

**Bad** :angry:

```python
import sqlite3
import smtplib


class OrderService:                                    # high-level policy
    def place_order(self, order: Order) -> None:
        connection = sqlite3.connect('shop.db')        # low-level detail
        connection.execute(
            'INSERT INTO orders VALUES (?, ?)', (order.id, str(order.total))
        )
        connection.commit()

        smtplib.SMTP('localhost').sendmail(            # another low-level detail
            'shop@example.com', order.customer_email, 'Thanks for your order'
        )
```

The business rule ("placing an order stores it and confirms it") is welded to SQLite and SMTP. You cannot
test it without both, you cannot move to Postgres without editing it, and the interesting logic is buried in
plumbing.

**Good** :smiley:

```python
from typing import Protocol


class OrderRepository(Protocol):            # abstraction owned by the policy
    def add(self, order: Order) -> None: ...


class Notifier(Protocol):
    def order_confirmed(self, order: Order) -> None: ...


class OrderService:                         # depends only on the abstractions
    def __init__(self, orders: OrderRepository, notifier: Notifier) -> None:
        self._orders = orders
        self._notifier = notifier

    def place_order(self, order: Order) -> None:
        self._orders.add(order)
        self._notifier.order_confirmed(order)
```

```python
class SqliteOrderRepository:                # detail, implements the abstraction
    def __init__(self, path: str) -> None:
        self._path = path

    def add(self, order: Order) -> None:
        with sqlite3.connect(self._path) as connection:
            connection.execute(
                'INSERT INTO orders VALUES (?, ?)', (order.id, str(order.total))
            )


class EmailNotifier:
    def __init__(self, smtp_host: str) -> None:
        self._smtp_host = smtp_host

    def order_confirmed(self, order: Order) -> None:
        smtplib.SMTP(self._smtp_host).sendmail(
            'shop@example.com', order.customer_email, 'Thanks for your order'
        )
```

The wiring happens once, at the edge of the application:

```python
def main() -> None:
    service = OrderService(
        orders=SqliteOrderRepository('shop.db'),
        notifier=EmailNotifier('localhost'),
    )
    service.place_order(build_order())
```

This is **dependency injection** — the mechanism — in service of **dependency inversion** — the design goal.

#### What it buys you

Testing becomes trivial, and the test reads like a specification:

```python
class InMemoryOrderRepository:
    def __init__(self) -> None:
        self.orders: list[Order] = []

    def add(self, order: Order) -> None:
        self.orders.append(order)


class RecordingNotifier:
    def __init__(self) -> None:
        self.confirmed: list[Order] = []

    def order_confirmed(self, order: Order) -> None:
        self.confirmed.append(order)


def test_placing_an_order_stores_and_confirms_it() -> None:
    orders, notifier = InMemoryOrderRepository(), RecordingNotifier()
    service = OrderService(orders, notifier)

    order = Order(id='1', total=Decimal('100'), customer_email='a@b.com')
    service.place_order(order)

    assert orders.orders == [order]
    assert notifier.confirmed == [order]
```

No database, no mail server, no mocking library, no sleeping for network timeouts — and the test breaks only
when the *behaviour* changes.

#### Doing it the Pythonic way

You rarely need a dependency injection framework. Python gives you lighter options:

```python
# 1. Constructor injection (the default choice)
class Report:
    def __init__(self, clock: Callable[[], datetime] = datetime.now) -> None:
        self._clock = clock


# 2. Pass a function, not an object with one method
def retry(action: Callable[[], T], attempts: int = 3) -> T: ...


# 3. Inject at the call site with a default
def render(template: str, *, loader: Loader = FileLoader()) -> str: ...
```

Two warnings:

- **Do not invert everything.** Abstracting a dependency that will never change (the `math` module, a
  dataclass you own) adds indirection and buys nothing. Invert at the boundaries: I/O, time, randomness,
  third-party services, anything slow or non-deterministic.
- **The abstraction belongs to the caller.** `OrderRepository` lives with `OrderService`, not with the
  SQLite code. If you put it in the database module, the dependency arrow never actually inverted.

> **Rule of thumb:** the parts of your system that encode business rules should be the parts that are
> hardest to break and easiest to test. If they import a driver, they are neither.

**[⬆ back to top](#table-of-contents)**

