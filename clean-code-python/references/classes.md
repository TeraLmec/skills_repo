## **Classes**

## Table of Contents

- [Classes](#classes)
- [Classes should be small](#classes-should-be-small)
- [Cohesion](#cohesion)
- [Organising for change](#organising-for-change)
- [Depend on abstractions, not details](#depend-on-abstractions-not-details)


---

Classes follow the same rules as functions, one size up: they should be **small**, they should do **one
thing**, and their name should say what that thing is.

### Classes should be small

For a function, we count lines. For a class, we count **responsibilities**.

The first hint is the name. If you cannot name a class without using `Manager`, `Processor`, `Super` or
`And`, it probably has more than one responsibility. The second hint is the description: if the sentence
describing the class needs an "and", split it.

**Bad** :angry:

```python
class SuperDashboard:
    def get_last_focused_component(self): ...
    def set_last_focused(self, component): ...
    def get_major_version_number(self) -> int: ...
    def get_minor_version_number(self) -> int: ...
    def get_build_number(self) -> int: ...
    def refresh_widgets(self): ...
    def persist_layout(self): ...
    def send_telemetry(self): ...
```

Version numbers, focus tracking, rendering, persistence and telemetry are five reasons for this class to
change (see the [Single Responsibility Principle](single-responsibility.md#single-responsibility-principle)).

**Good** :smiley:

```python
@dataclass(frozen=True)
class Version:
    major: int
    minor: int
    build: int

    def __str__(self) -> str:
        return f'{self.major}.{self.minor}.{self.build}'


class FocusTracker:
    def last_focused(self) -> Component: ...
    def record_focus(self, component: Component) -> None: ...


class Dashboard:
    def refresh_widgets(self) -> None: ...
```

A system with many small, single-purpose classes has no more moving parts than a system with a few large
ones — it has the same amount of logic, better organised. The difference is that you only need to understand
the handful of classes involved in the change you are making.

### Cohesion

A class is **cohesive** when its methods use its attributes. In a maximally cohesive class, every method
touches every attribute; that is rare and not a goal, but the trend matters.

Low cohesion is a split waiting to happen: if methods `a` and `b` only touch `self.x`, and methods `c` and
`d` only touch `self.y`, you have two classes sharing a namespace.

**Bad** :angry:

```python
class ReportEngine:
    def __init__(self, rows, smtp_host):
        self.rows = rows            # used only by the two methods below
        self.smtp_host = smtp_host  # used only by send()

    def to_csv(self) -> str:
        return '\n'.join(','.join(map(str, row)) for row in self.rows)

    def row_count(self) -> int:
        return len(self.rows)

    def send(self, body: str, to: str) -> None:
        smtplib.SMTP(self.smtp_host).sendmail('reports@example.com', to, body)
```

**Good** :smiley:

```python
class Report:
    def __init__(self, rows: list[list[str]]) -> None:
        self.rows = rows

    def to_csv(self) -> str:
        return '\n'.join(','.join(row) for row in self.rows)

    def row_count(self) -> int:
        return len(self.rows)


class Mailer:
    def __init__(self, smtp_host: str) -> None:
        self._smtp_host = smtp_host

    def send(self, body: str, to: str) -> None:
        smtplib.SMTP(self._smtp_host).sendmail('reports@example.com', to, body)
```

A class whose cohesion has dropped to zero — no shared state at all — is not a class. In Python that is a
module of functions, and you should write it as one instead of using a class as a namespace:

**Bad** :angry:

```python
class MathUtils:
    @staticmethod
    def mean(values: list[float]) -> float:
        return sum(values) / len(values)
```

**Good** :smiley:

```python
# statistics.py
def mean(values: list[float]) -> float:
    return sum(values) / len(values)
```

### Organising for change

The goal of class design is not to prevent change — it is to make change **local**. When a new requirement
arrives, you want to add a class rather than open several existing ones.

Compare a class that must be edited for every new query type:

**Bad** :angry:

```python
class Sql:
    def create(self, table, columns): ...
    def insert(self, table, fields): ...
    def select_all(self, table): ...
    def select_with_criteria(self, table, criteria): ...
    def _prepare_columns(self, columns): ...      # used by two of the above
```

with a design where a new query is a new class:

**Good** :smiley:

```python
class Sql(ABC):
    @abstractmethod
    def generate(self) -> str: ...


class CreateSql(Sql):
    def generate(self) -> str: ...


class SelectSql(Sql):
    def generate(self) -> str: ...


class SelectWithCriteriaSql(Sql):
    def generate(self) -> str: ...
```

Adding `UpdateSql` now touches no existing code, which is the
[Open/Closed Principle](open-closed.md#openclosed-principleocp) in practice, and each class is independently testable.

### Depend on abstractions, not details

Classes should depend on interfaces rather than concrete implementations, so that the things most likely to
change (a database driver, an HTTP API, the clock) sit behind a boundary you control.

**Bad** :angry:

```python
class PortfolioValuer:
    def total(self) -> Decimal:
        prices = requests.get('https://api.example.com/prices').json()   # hard-wired
        ...
```

That class cannot be tested without a network, and it cannot be reused with a different price source.

**Good** :smiley:

```python
class PriceSource(Protocol):
    def price_of(self, symbol: str) -> Decimal: ...


class PortfolioValuer:
    def __init__(self, prices: PriceSource) -> None:
        self._prices = prices

    def total(self, holdings: dict[str, int]) -> Decimal:
        return sum(
            (self._prices.price_of(symbol) * quantity for symbol, quantity in holdings.items()),
            start=Decimal('0'),
        )
```

Tests inject a fixed price list; production injects the HTTP client. This is the
[Dependency Inversion Principle](dependency-inversion.md#dependency-inversion-principle), and it is what makes the rest of the SOLID
principles usable.

**[⬆ back to top](#table-of-contents)**

