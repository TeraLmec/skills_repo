### Interface Segregation Principle

## Table of Contents

- [Interface Segregation Principle](#interface-segregation-principle)
- [The violation](#the-violation)
- [A practical example: printers](#a-practical-example-printers)
- [ISP in everyday Python](#isp-in-everyday-python)


---

> **No client should be forced to depend on methods it does not use.**

Fat interfaces couple everybody to everything. When one class declares twenty methods and each caller uses
two of them, a change to any of the twenty forces every caller to be re-checked, re-compiled and re-tested.

#### The violation

**Bad** :angry:

```python
from abc import ABC, abstractmethod


class Worker(ABC):
    @abstractmethod
    def work(self) -> None: ...

    @abstractmethod
    def eat(self) -> None: ...

    @abstractmethod
    def sleep(self) -> None: ...


class HumanWorker(Worker):
    def work(self) -> None: ...
    def eat(self) -> None: ...
    def sleep(self) -> None: ...


class RobotWorker(Worker):
    def work(self) -> None: ...

    def eat(self) -> None:
        raise NotImplementedError('Robots do not eat')     # forced to implement

    def sleep(self) -> None:
        raise NotImplementedError('Robots do not sleep')
```

`RobotWorker` is dragged into a contract that has nothing to do with it, and the `NotImplementedError`s are
also an [LSP](liskov-substitution.md#liskov-substitution-principle) violation — the two principles usually break together.

**Good** :smiley: — many small, role-based interfaces

```python
from typing import Protocol


class Workable(Protocol):
    def work(self) -> None: ...


class Feedable(Protocol):
    def eat(self) -> None: ...


class Sleepable(Protocol):
    def sleep(self) -> None: ...


class HumanWorker:
    def work(self) -> None: ...
    def eat(self) -> None: ...
    def sleep(self) -> None: ...


class RobotWorker:
    def work(self) -> None: ...


def run_shift(workers: list[Workable]) -> None:      # asks for exactly what it needs
    for worker in workers:
        worker.work()


def lunch_break(staff: list[Feedable]) -> None:
    for member in staff:
        member.eat()
```

`run_shift` accepts both humans and robots; `lunch_break` accepts only humans, and the type checker enforces
it. Neither class had to inherit from anything.

#### A practical example: printers

**Bad** :angry:

```python
class MultiFunctionDevice(ABC):
    @abstractmethod
    def print_document(self, doc: Document) -> None: ...

    @abstractmethod
    def scan(self) -> Document: ...

    @abstractmethod
    def fax(self, doc: Document, number: str) -> None: ...


class CheapPrinter(MultiFunctionDevice):
    def print_document(self, doc: Document) -> None: ...
    def scan(self) -> Document:
        raise NotImplementedError
    def fax(self, doc: Document, number: str) -> None:
        raise NotImplementedError
```

**Good** :smiley:

```python
class Printer(Protocol):
    def print_document(self, doc: Document) -> None: ...


class Scanner(Protocol):
    def scan(self) -> Document: ...


class Fax(Protocol):
    def fax(self, doc: Document, number: str) -> None: ...


class CheapPrinter:
    def print_document(self, doc: Document) -> None: ...


class OfficeAllInOne:
    """Satisfies Printer, Scanner and Fax without inheriting from any of them."""
    def print_document(self, doc: Document) -> None: ...
    def scan(self) -> Document: ...
    def fax(self, doc: Document, number: str) -> None: ...
```

#### ISP in everyday Python

You do not need `ABC` or `Protocol` to apply this principle — the idea is broader than interfaces:

- **Function parameters.** Take the narrowest type that works: `Iterable[str]` rather than `list[str]` if
  you only iterate; `Mapping` rather than `dict` if you only read.

  ```python
  def total(prices: Iterable[Decimal]) -> Decimal:      # accepts lists, tuples, generators, sets
      return sum(prices, start=Decimal('0'))
  ```

- **Configuration objects.** Do not pass the whole `Settings` object into a class that needs one timeout.
  Pass the timeout.

- **Test doubles as a signal.** If faking a dependency in a test requires stubbing eight methods you never
  call, the interface is too fat. Test pain is design feedback.

> **Rule of thumb:** the size of an interface should be decided by its *clients*, not by its implementation.
> Many small interfaces are easier to satisfy, easier to fake and easier to keep stable than one large one.

**[⬆ back to top](#table-of-contents)**

