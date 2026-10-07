## **Testing**

## Table of Contents

- [Testing](#testing)
- [The three laws of TDD](#the-three-laws-of-tdd)
- [One assert (one concept) per test](#one-assert-one-concept-per-test)
- [Arrange, Act, Assert](#arrange-act-assert)
- [F.I.R.S.T.](#first)
- [Test behaviour, not implementation](#test-behaviour-not-implementation)
- [Prefer fakes to mocks](#prefer-fakes-to-mocks)
- [Parametrise instead of copy-pasting](#parametrise-instead-of-copy-pasting)


---

Tests are not a chore you perform after the real work. They are the thing that makes the real work
*changeable*: without them, every refactoring in this guide is a gamble.

> **Test code is just as important as production code.** It is not a second-class citizen. It requires
> thought, design and care. It must be kept as clean as production code.

### The three laws of TDD

1. You may not write production code until you have written a failing test.
2. You may not write more of a test than is sufficient to fail (and not compiling counts as failing).
3. You may not write more production code than is sufficient to pass the currently failing test.

Working this way keeps tests and code in lockstep and guarantees that every line you ship is covered by
something that would have caught its absence.

### One assert (one concept) per test

A test that verifies five things fails on the first one and tells you nothing about the other four.

**Bad** :angry:

```python
def test_account():
    account = Account(100)
    account.deposit(50)
    assert account.balance == 150
    account.withdraw(30)
    assert account.balance == 120
    with pytest.raises(InsufficientFunds):
        account.withdraw(1000)
    assert account.is_active
```

**Good** :smiley:

```python
def test_deposit_increases_balance():
    account = Account(100)
    account.deposit(50)
    assert account.balance == 150


def test_withdrawal_decreases_balance():
    account = Account(100)
    account.withdraw(30)
    assert account.balance == 70


def test_withdrawing_more_than_the_balance_is_rejected():
    account = Account(100)
    with pytest.raises(InsufficientFunds):
        account.withdraw(1000)
```

Each name is a sentence about the system, and the suite doubles as documentation. Notice that the test names
are long — that is correct. A test name has one job: telling you what broke, from the failure output alone.

### Arrange, Act, Assert

Give every test the same three-part shape, and the reader never has to work out which part is which.

```python
def test_shipping_is_free_above_the_threshold():
    # Arrange
    cart = Cart(items=[Item('book', Decimal('60'))])

    # Act
    cost = shipping_cost(cart)

    # Assert
    assert cost == Decimal('0')
```

The `pytest` equivalent of a shared "Arrange" is a fixture:

```python
@pytest.fixture
def account() -> Account:
    return Account(balance=Decimal('100'))


def test_deposit_increases_balance(account: Account) -> None:
    account.deposit(Decimal('50'))
    assert account.balance == Decimal('150')
```

### F.I.R.S.T.

Clean tests follow five rules:

- **Fast.** Slow tests do not get run, and tests that do not get run do not prevent bugs.
- **Independent.** No test may depend on another running first, or on the order of the suite.
- **Repeatable.** The same result on your laptop, in CI and on a plane with no network. That means no
  reliance on the real clock, real randomness or a shared database — inject those (see
  [Dependency Inversion](dependency-inversion.md#dependency-inversion-principle)).
- **Self-validating.** A test passes or fails. It does not print output for a human to inspect.
- **Timely.** Written just before the production code, while the design is still soft.

**Bad** :angry: — not repeatable

```python
def test_invoice_is_due_in_30_days():
    invoice = Invoice(issued_at=datetime.now())
    assert invoice.due_date == datetime.now() + timedelta(days=30)   # flaky
```

**Good** :smiley:

```python
def test_invoice_is_due_in_30_days():
    issued = datetime(2024, 1, 1)
    invoice = Invoice(issued_at=issued)
    assert invoice.due_date == datetime(2024, 1, 31)
```

### Test behaviour, not implementation

A test that reaches into private attributes or asserts on the exact sequence of internal calls will break
every time you refactor — which is exactly when you need it to keep working.

**Bad** :angry:

```python
def test_register_hashes_password(mocker):
    hasher = mocker.patch('app.hashlib.sha256')       # coupled to the algorithm
    register('a@b.com', 'secret123', users, mailer)
    hasher.assert_called_once()
```

**Good** :smiley:

```python
def test_registered_user_can_log_in_with_their_password():
    users = InMemoryUserRepository()
    register('a@b.com', 'secret123', users, RecordingMailer())
    assert authenticate('a@b.com', 'secret123', users) is not None


def test_the_raw_password_is_never_stored():
    users = InMemoryUserRepository()
    register('a@b.com', 'secret123', users, RecordingMailer())
    assert 'secret123' not in users.get('a@b.com').password_hash
```

Both tests would survive a switch from SHA-256 to bcrypt; the first one would not.

### Prefer fakes to mocks

A hand-written fake is usually clearer than a stack of `patch` calls, and it fails loudly when the real
interface changes:

```python
class InMemoryUserRepository:
    def __init__(self) -> None:
        self._users: dict[str, User] = {}

    def add(self, user: User) -> None:
        self._users[user.email] = user

    def get(self, email: str) -> User | None:
        return self._users.get(email)
```

Heavy mocking is often a design smell rather than a testing technique: if a unit cannot be tested without
patching five modules, it depends on five things it should not know about.

### Parametrise instead of copy-pasting

```python
@pytest.mark.parametrize(
    'amount, expected',
    [
        (Decimal('0'), Decimal('0')),
        (Decimal('100'), Decimal('16')),
        (Decimal('1000'), Decimal('160')),
    ],
)
def test_value_added_tax(amount: Decimal, expected: Decimal) -> None:
    assert value_added_tax(amount) == expected
```

> **Coverage is a floor, not a goal.** 100% coverage of trivial assertions proves nothing; a smaller suite
> that pins down real behaviour at the boundaries is worth far more.

**[⬆ back to top](#table-of-contents)**

