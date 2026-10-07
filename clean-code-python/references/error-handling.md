## **Error Handling**

## Table of Contents

- [Error Handling](#error-handling)
- [Use exceptions rather than return codes](#use-exceptions-rather-than-return-codes)
- [Separate the happy path from the error path](#separate-the-happy-path-from-the-error-path)
- [Never swallow an exception](#never-swallow-an-exception)
- [Provide context with your exceptions](#provide-context-with-your-exceptions)
- [Define exception classes for the caller's needs](#define-exception-classes-for-the-callers-needs)
- [Don't return None, don't pass None](#dont-return-none-dont-pass-none)
- [Fail fast](#fail-fast)
- [Use finally and context managers for cleanup](#use-finally-and-context-managers-for-cleanup)


---

Error handling is important, but if it obscures logic, it is wrong. Things go wrong in every program; the
question is whether the code that deals with it hides the code that does the work.

### Use exceptions rather than return codes

Returning a status code forces every caller to check it, and nothing stops them forgetting.

**Bad** :angry:

```python
def delete_page(page_id: str) -> int:
    if not exists(page_id):
        return E_NOT_FOUND
    if not can_delete(page_id):
        return E_FORBIDDEN
    ...
    return OK


code = delete_page(page_id)          # nothing forces you to look at this
if code == OK:
    ...
```

**Good** :smiley:

```python
def delete_page(page_id: str) -> None:
    if not exists(page_id):
        raise PageNotFound(page_id)
    if not can_delete(page_id):
        raise DeletionForbidden(page_id)
    ...
```

The happy path is now the only path in the function body, and a caller who ignores the failure gets a loud
traceback instead of silent corruption.

### Separate the happy path from the error path

`try` blocks are transactions: what follows `except` should leave the program in a consistent state
regardless of what the block managed to do. Extract the bodies so the structure is visible.

**Bad** :angry:

```python
def report(path: str) -> None:
    try:
        handle = open(path)
        rows = [line.split(',') for line in handle]
        total = sum(Decimal(row[2]) for row in rows)
        print(f'Total: {total}')
        handle.close()
    except Exception as error:
        print(error)
```

**Good** :smiley:

```python
def report(path: str) -> None:
    try:
        print(f'Total: {total_of(path)}')
    except FileNotFoundError:
        logger.error('No such report file: %s', path)
        raise


def total_of(path: str) -> Decimal:
    with open(path) as handle:
        rows = (line.split(',') for line in handle)
        return sum((Decimal(row[2]) for row in rows), start=Decimal('0'))
```

### Never swallow an exception

**Bad** :angry:

```python
try:
    charge_customer(order)
except Exception:
    pass                     # the customer was never charged; nobody will ever know
```

A bare `except: pass` is the single most expensive line in this guide. If you truly can continue, say why in
a comment, catch the *specific* exception, and log it.

**Good** :smiley:

```python
try:
    send_analytics_event(order)
except AnalyticsUnavailable:
    # Analytics is best-effort: a dropped event must never block a sale.
    logger.warning('Analytics event dropped for order %s', order.id)
```

Related: `except Exception` also catches `KeyboardInterrupt`'s siblings and programming errors like
`AttributeError`, turning a bug into a mystery. Catch the narrowest exception that describes what you can
actually handle.

### Provide context with your exceptions

An exception should say what failed and what was being attempted.

**Bad** :angry:

```python
raise ValueError('invalid')
```

**Good** :smiley:

```python
raise InvalidTradeFile(
    f'Row {row_number} of {path} has {len(fields)} fields, expected 5'
)
```

When you re-raise, keep the original cause so the traceback tells the whole story:

```python
try:
    response = client.get(url)
except httpx.HTTPError as error:
    raise PriceSourceUnavailable(url) from error      # 'from' preserves the chain
```

### Define exception classes for the caller's needs

Group errors by what the caller will *do* about them, not by which library produced them.

**Bad** :angry:

```python
try:
    port = acme_port.open()
except DeviceResponseException as error:
    log(error)
except ATM1212UnlockedException as error:
    log(error)
except GMXError as error:
    log(error)
```

**Good** :smiley:

```python
class PortDeviceFailure(Exception):
    """Anything that goes wrong while talking to a port device."""


class AcmePort:
    def __init__(self, port: int) -> None:
        self._inner = ACMEPort(port)

    def open(self) -> None:
        try:
            self._inner.open()
        except (DeviceResponseException, ATM1212UnlockedException, GMXError) as error:
            raise PortDeviceFailure() from error
```

Wrapping a third-party API like this is almost always worth it: it collapses a family of errors into one
concept, and it keeps the vendor's names from leaking into your whole codebase (see
[Dependency Inversion](dependency-inversion.md#dependency-inversion-principle)).

### Don't return None, don't pass None

Returning `None` pushes a check onto every caller, and the one who forgets gets an `AttributeError` far from
the cause.

**Bad** :angry:

```python
def find_employee(employee_id: str) -> Employee | None:
    ...

total = find_employee('123').salary       # AttributeError one day
```

**Good** :smiley:

```python
def employees_in(department: str) -> list[Employee]:
    return self._by_department.get(department, [])      # empty list, never None


for employee in employees_in('sales'):                   # no check needed
    ...
```

Where absence is genuinely a valid outcome, make it explicit in the type (`Employee | None`) so the type
checker forces the caller to handle it — or raise, if absence is an error.

Passing `None` into a function is worse, because no amount of defensive code in the callee fixes a caller
that should not have done it. Prefer a default, an empty collection, or a separate function.

### Fail fast

Validate at the boundary, then trust your own data.

```python
@dataclass(frozen=True)
class Percentage:
    value: Decimal

    def __post_init__(self) -> None:
        if not 0 <= self.value <= 100:
            raise ValueError(f'Percentage must be between 0 and 100, got {self.value}')
```

Once a `Percentage` exists, no function that takes one ever has to check it again. This is how types replace
defensive programming.

### Use `finally` and context managers for cleanup

**Bad** :angry:

```python
connection = pool.acquire()
result = query(connection)        # if this raises, the connection leaks
pool.release(connection)
return result
```

**Good** :smiley:

```python
with pool.acquire() as connection:
    return query(connection)
```

If a resource you own does not provide a context manager, write one — `contextlib.contextmanager` makes it
three lines:

```python
from contextlib import contextmanager


@contextmanager
def acquired(pool: Pool):
    connection = pool.acquire()
    try:
        yield connection
    finally:
        pool.release(connection)
```

**[⬆ back to top](#table-of-contents)**

