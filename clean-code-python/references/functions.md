## **Functions**

## Table of Contents

- [Functions](#functions)
- [Small](#small)
- [Do-One-Thing](#do-one-thing)
- [One-level-of-Abstraction](#one-level-of-abstraction)
- [The Stepdown Rule](#the-stepdown-rule)
- [Avoid Conditionals](#avoid-conditionals)
- [Use-Descriptive-Names](#use-descriptive-names)
- [Function-Arguments](#function-arguments)
- [Common forms](#common-forms)
- [Flag arguments are ugly](#flag-arguments-are-ugly)
- [Mutable default arguments](#mutable-default-arguments)
- [Prefer keyword arguments at the call site](#prefer-keyword-arguments-at-the-call-site)
- [*args and **kwargs](#args-and-kwargs)
- [Avoid-Side-Effects](#avoid-side-effects)
- [Pure Functions](#pure-functions)
- [1. Niladic-Functions](#1-niladic-functions)
- [2. Argument Mutation](#2-argument-mutation)
- [3. Exceptions](#3-exceptions)
- [4. I/O](#4-io)
- [Command-Query-Separation](#command-query-separation)
- [Recognising a violation](#recognising-a-violation)
- [When to break the rule](#when-to-break-the-rule)
- [Don't Repeat Yourself (DRY)](#dont-repeat-yourself-dry)


### **Small**

---

The first rule of functions is that they should be small. The second rule is that they should be smaller
than that.

There is no magic number, but there is a reliable signal: a function you can take in **without scrolling**
and **without holding state in your head** is small enough. In practice that lands somewhere between three
and fifteen lines for most Python.

Length is a symptom, not the disease. A long function is long because it is doing several things, and each
of those things is a concept that deserves a name. Extracting them is not busywork; it is how the concepts
become visible.

**Bad** :angry:

```python
def send_invoices(customers, tax_rate, smtp_host, dry_run=False):
    results = []
    for customer in customers:
        if not customer.get('active'):
            continue
        subtotal = 0.0
        for line in customer['orders']:
            if line['status'] == 'shipped':
                subtotal += line['price'] * line['quantity']
        if subtotal == 0:
            continue
        tax = subtotal * tax_rate
        total = subtotal + tax
        body = f"Dear {customer['name']},\n\nYou owe {total:.2f} ({tax:.2f} tax).\n"
        if not dry_run:
            server = smtplib.SMTP(smtp_host)
            server.sendmail('billing@example.com', customer['email'], body)
            server.quit()
        results.append((customer['email'], total))
    return results
```

To review that function you have to keep four unrelated jobs in your head at once: filtering customers,
summing orders, formatting an email and talking to an SMTP server. Note also the flag argument (`dry_run`),
which is a confession that the function does two things.

**Good** :smiley:

```python
from decimal import Decimal
from typing import Iterable, List


def send_invoices(customers: Iterable[Customer], tax_rate: Decimal, mailer: Mailer) -> List[Invoice]:
    invoices = [invoice_for(customer, tax_rate) for customer in customers if is_billable(customer)]
    for invoice in invoices:
        mailer.send(invoice.recipient, render_invoice(invoice))
    return invoices


def is_billable(customer: Customer) -> bool:
    return customer.is_active and amount_due(customer) > 0


def amount_due(customer: Customer) -> Decimal:
    return sum(
        (order.price * order.quantity for order in customer.orders if order.is_shipped),
        start=Decimal('0'),
    )


def invoice_for(customer: Customer, tax_rate: Decimal) -> Invoice:
    subtotal = amount_due(customer)
    return Invoice(recipient=customer.email, subtotal=subtotal, tax=subtotal * tax_rate)


def render_invoice(invoice: Invoice) -> str:
    return f'Dear customer,\n\nYou owe {invoice.total:.2f} ({invoice.tax:.2f} tax).\n'
```

Every function now fits on a screen, and the `dry_run` flag disappeared: passing a different `Mailer` (a
real one in production, a recording fake in tests) covers the same need without a branch.

Two Python-specific notes:

- **Blocks inside `if`, `for` and `while` should usually be one line long** — ideally a call to a well named
  function. That keeps nesting shallow, and nesting is what makes a function hard to read.
- **Indentation should rarely exceed two levels.** Three or more nested blocks means there is a function
  hiding in there. Guard clauses and early `return`s flatten most of them.

**Bad** :angry:

```python
def grade(student):
    if student is not None:
        if student.is_enrolled:
            if student.marks:
                return sum(student.marks) / len(student.marks)
    return None
```

**Good** :smiley:

```python
def grade(student: Student | None) -> float | None:
    if student is None or not student.is_enrolled or not student.marks:
        return None
    return sum(student.marks) / len(student.marks)
```

**[⬆ back to top](#table-of-contents)**

### Do-One-Thing

---

> **Functions should do one thing. They should do it well. They should do it only.**

The hard part is agreeing on what "one thing" means, because every function can be described as several
smaller steps. Two tests work well in practice:

1. **Can you extract another function from it whose name is not merely a restatement of its body?** If yes,
   the function was doing more than one thing.
2. **Can you describe it in one sentence with no "and", "then" or "or"?** "Validate the trade *and* store
   it" is two things.

**Bad** :angry:

```python
def register(email: str, password: str) -> str:
    if '@' not in email:
        raise ValueError('bad email')
    if len(password) < 8:
        raise ValueError('weak password')

    hashed = hashlib.sha256(password.encode()).hexdigest()

    connection = sqlite3.connect('users.db')
    connection.execute('INSERT INTO users VALUES (?, ?)', (email, hashed))
    connection.commit()

    smtplib.SMTP('localhost').sendmail(
        'noreply@example.com', email, 'Subject: Welcome!\n\nThanks for joining.'
    )
    return email
```

This function validates, hashes, persists **and** sends mail. It has four reasons to change, it cannot be
tested without a database and an SMTP server, and there is no way to reuse the validation on its own.

**Good** :smiley:

```python
def register(email: str, password: str, users: UserRepository, mailer: Mailer) -> User:
    validate_credentials(email, password)
    user = User(email=email, password_hash=hash_password(password))
    users.add(user)
    mailer.send_welcome(user)
    return user


def validate_credentials(email: str, password: str) -> None:
    if '@' not in email:
        raise InvalidEmail(email)
    if len(password) < MIN_PASSWORD_LENGTH:
        raise WeakPassword()


def hash_password(password: str) -> str:
    return hashlib.sha256(password.encode()).hexdigest()
```

`register` still mentions four steps, but it no longer *performs* them: it is one level of policy, delegating
to one level of detail. That is exactly the shape the next section is about.

Sections within a function are the clearest sign of a violation. If you find yourself writing
`# --- validation ---` and `# --- persistence ---` comments, you have found your extraction points; the
comment is trying to tell you the function's name.

**[⬆ back to top](#table-of-contents)**

### One-level-of-Abstraction

---

Mixing levels of abstraction inside one function forces the reader to constantly change altitude: one line
is about business policy, the next is about string slicing, the one after is about socket timeouts. Each
switch costs attention.

**Bad** :angry:

```python
def publish_report(rows: list[dict]) -> None:
    html = '<table>'
    for row in rows:                                    # low level: string building
        html += '<tr>' + ''.join(f'<td>{v}</td>' for v in row.values()) + '</tr>'
    html += '</table>'

    if datetime.now().weekday() >= 5:                   # high level: business rule
        return

    with open('/var/www/report.html', 'w') as handle:    # low level: file system
        handle.write(html)
    notify_subscribers()                                # high level again
```

**Good** :smiley:

```python
def publish_report(rows: list[dict]) -> None:
    if is_weekend():
        return
    write_report(render_html(rows))
    notify_subscribers()


def render_html(rows: list[dict]) -> str:
    body = ''.join(render_row(row) for row in rows)
    return f'<table>{body}</table>'


def render_row(row: dict) -> str:
    cells = ''.join(f'<td>{value}</td>' for value in row.values())
    return f'<tr>{cells}</tr>'


def write_report(html: str) -> None:
    Path('/var/www/report.html').write_text(html)
```

`publish_report` now reads like a summary of the process. Each function it calls is one step lower, and each
of those is again internally consistent.

#### The Stepdown Rule

Code reads best as a top-down narrative. Every function should be followed by those at the next level of
abstraction, so that reading the module is like reading a set of "TO" paragraphs:

> **To** publish a report, we skip weekends, render the rows to HTML, write the file and notify subscribers.
> &nbsp;&nbsp;**To** render rows to HTML, we render each row and wrap them in a table.
> &nbsp;&nbsp;&nbsp;&nbsp;**To** render a row, we wrap each value in a cell.

Following the stepdown rule means the reader can stop at any depth. Someone reviewing a business rule reads
the first function and leaves; someone debugging the markup keeps descending.

**[⬆ back to top](#table-of-contents)**

### Avoid Conditionals

---

Let us meet Joe. Joe is a junior web developer who works at a certain company in Nairobi. Joe's company has got a new client who wants Joe's company to build him an application to manage his bank.

The client specifies that this application will manage user bank accounts. Joe organizes a meeting with the client and they agree to meet so that Joe can collect the client's business needs. Let us watch Joe as he puts his OOP programming skills to work.

After their meeting, they agree that the user account will have the following attributes and behaviour.

**Account class structure**

The class tables in this guide spell out the UML members and their visibility. In UML signatures,
`String`, `Real`, `Boolean` and `void` correspond to Python's `str`, `float`, `bool` and `None`.
Private and protected visibility describe the design intent; Python does not enforce access restrictions.

| Member | Visibility | Type or signature |
| --- | --- | --- |
| `acc_name` | Private | `String` |
| `acc_number` | Private | `String` |
| `amount` | Private | `Real` |
| `__eq__` | Public | `__eq__(Account): Boolean` |
| `__str__` | Public | `__str__(): String` |
| `add_interest` | Public | `add_interest(): void` |
| `deposit` | Public | `deposit(Real): void` |
| `get_balance` | Public | `get_balance(): Real` |
| `get_loan` | Public | `get_loan(Real): Boolean` |
| `has_enough_collateral` | Private | `has_enough_collateral(Real): Boolean` |
| `withdraw` | Public | `withdraw(Real): void` |

The implementation below calls `acc_name` simply `name` and prefixes the collateral helper with `_`.

The code below provides the implementation details of this class.

```python
class Account:
    def __init__(self, acc_number: str, amount: float, name: str):
        self.acc_number = acc_number
        self.amount = amount
        self.name = name

    def get_balance(self) -> float:
        return self.amount

    def __eq__(self, other: Account) -> bool:
        if isinstance(other, Account):
            return self.acc_number == other.acc_number

    def deposit(self, amount: float) -> bool:
        if amount > 0:
            self.amount += amount

    def withdraw(self, amount: float) -> None:
        if (amount > 0) and (amount <= self.amount):
            self.amount -= amount

    def _has_enough_collateral(self, loan: float) -> bool:
        if loan < self.amount / 2:
            return True

    def __str__(self) -> str:
        return f'Account acc number : {self.acc_number} amount : {self.amount}'

    def add_interest(self) -> None:
        self.deposit(0.1 * self.amount)

    def get_loan(self, amount : float) -> bool:
        if self._has_enough_collateral(amount):
            return True
        else:
            return False
```

The application is a success and after a month, the client comes back to Joe asking for more features. The client says that he now wants the application to work with more than one type of account. The application should now process SavingsAccount and CheckingAccount accounts. The difference between them is outlined below.

- When authorizing a loan, a checking account needs a
  balance of two thirds the loan amount, whereas savings accounts require only one half the loan amount.

- The bank gives periodic interest to savings accounts but not checking accounts.

- The representation of an account will return
  “Savings Account” or “Checking Account,” as appropriate.

Joe rolls up his sleeves and starts to make modifications to the original Account class to introduce the new features. Below is his approach.

**Bad** :angry:

```python
class Account:
    def __init__(self, acc_number: str, amount: float, name: str, type : int):
        self.acc_number = acc_number
        self.amount = amount
        self.name = name
        self.type = type

    def _has_enough_collateral(self, loan: float) -> bool:
        if self.type == 1:
            return self.amount >= loan / 2;
        elif selt.type == 2:
            return self.amount >= 2 * loan / 3;
        else:
            return False

    def __str__(self) -> str:
        if self.type == 1:
            return ' SavingsAccount'
        elif self.type == 2:
            return 'CheckingAccount'
        else:
            return 'InvalidAccount'

    def add_interest(self) -> None:
        if self.type == 1: self.deposit(0.1 * self.amount)


    def get_loan(self, amount : float) -> bool:
        True if self._has_enough_collateral(amount) else False

    #... other methods
```

> **Note:** We have only shown the methods that changed.

With this implementation, Joe is happy and he ships the app into production since it works as the client had wanted. But something has really gone wrong here.

The repeated conditionals in the preceding implementation are the maintenance problem:

| Method | Conditional behaviour |
| --- | --- |
| `_has_enough_collateral(loan)` | For `type == 1`, require a balance of at least `loan / 2`; for `type == 2`, require at least `2 * loan / 3`; otherwise return `False`. |
| `__str__()` | Return `SavingsAccount` for type 1, `CheckingAccount` for type 2, or `InvalidAccount` for any other type. |
| `add_interest()` | Deposit `0.1 * self.amount` only for type 1. |
| `get_loan(amount)` | Evaluate `True if self._has_enough_collateral(amount) else False`. This repeats a Boolean result; as written above, it also lacks a `return`. |

The constructor stores `acc_number`, `amount`, `name` and the new `type` tag. That tag controls behaviour
in several methods, so changing the set of account types affects several places in the same class.

The problem are these conditionals here. They work for now but they will cause a maintenance nightmare very soon. What will happen if the client comes back asking Joe to add more account types? Joe will have to open this class and add more IFs. What happens of the client asks him to delete some of the account types? He will open the same class and edit all Ifs again.

This class is now violating the **Single Responsibility Principle** and the **Open Closed Principle**. The class has more than one reason to change and still, it is not closed for modification and
these IFs may also run slow.

This smell is called the **Missing Hierarchy** smell.

> **Missing Hierarchy** <br/>
> This smell arises when a code segment uses conditional logic (typically in conjunction
> with “tagged types”) to explicitly manage variation in behavior where a hierarchy
> could have been created and used to encapsulate those variations.

To solve this problem, we will need to introduce an hierarchy of account types.
We will achieve this by creating a super abstract class Account and implement all the common methods but mark the account specific methods abstract.
Different account types can then inherit from this base class.

**Account inheritance hierarchy**

`SavingsAccount` and `CheckingAccount` both inherit from the abstract `Account` class. `Account` retains
the private attributes `acc_name: String`, `acc_number: String` and `amount: Real`, along with the complete
interface listed above. It implements the shared `deposit(Real): void`, `get_balance(): Real` and
`withdraw(Real): void` operations. Each subclass supplies the following account-specific operations:

| Operation | Visibility | `SavingsAccount` | `CheckingAccount` |
| --- | --- | --- | --- |
| `__eq__(Account): Boolean` | Public | Compare savings accounts | Compare checking accounts |
| `__str__(): String` | Public | Savings account representation | Checking account representation |
| `add_interest(): void` | Public | Add savings interest | No interest |
| `get_loan(Real): Boolean` | Public | Use savings collateral rules | Use checking collateral rules |
| `has_enough_collateral(Real): Boolean` | Private | Require half the loan amount | Require two thirds of the loan amount |

With this new approach, account specific methods will be implemented by subclasses and note that we will throw away those annoying IFs and replace them with polymorphism hence the **Replace Conditionals with Polymorphism** rule.

Below are the implementation of Account, SavingsAccount and CheckingAccount.

**Account abstract base class**

```python
from abc import ABC, abstractmethod


class Account(ABC):
    def __init__(self, acc_number: str, amount: float, name: str):
        self.acc_number = acc_number
        self.amount = amount
        self.name = name

    def get_balance(self) -> float:
        return self.amount

    def deposit(self, amount: float) -> bool:
        if amount > 0:
            self.amount += amount

    def withdraw(self, amount: float) -> None:
        if (amount > 0) and (amount <= self.amount):
            self.amount -= amount

    @abstractmethod
    def __eq__(self, other: 'SavingsAccount') -> bool:
        pass

    @abstractmethod
    def has_enough_collateral(self, loan: float) -> bool:
        pass

    @abstractmethod
    def __str__(self) -> str:
        pass

    @abstractmethod
    def get_loan(self) -> bool:
        pass

    @abstractmethod
    def add_interest(self) -> None:
        pass
```

The five methods marked `@abstractmethod` are the variation points; the constructor, balance lookup,
deposit and withdrawal remain shared. This transcription preserves the original example's signatures:
`deposit` is annotated `bool` but returns no value, `get_loan` omits the amount shown in the class interface,
and `has_enough_collateral` is named without the leading underscore used in some of the examples.
Those signatures need to agree with the subclasses in a runnable implementation.

**SavingsAccount class**

**Good** :smiley:

```python
class SavingAccount(Account):
    def __init__(self, acc_number: str, amount: float, name: str):
        Account.__init__(self, acc_number, amount, name)

    def __eq__(self, other: SavingsAccount) -> bool:
         if isinstance(other, SavingsAccount):
            return self.acc_number == other.acc_number


    def _has_enough_collateral(self, loan: float) -> bool:
        return self.amount >= loan / 2;

    def get_loan(self, amount : float) -> bool:
        return _has_enough_collateral(float)

    def __str__(self) -> str:
        return f'Saving Account acc number : {self.acc_number}'

    def add_interest(self) -> None:
        self.deposit(0.1 * self.amount)
```

**CheckingAccount class**

**Good** :smiley:

```python
class CheckingAccount(Account):
    def __init__(self, acc_number: str, amount: float, name: str):
        Account.__init__(self, acc_number, amount, name)

    def __eq__(self, other: SavingsAccount) -> bool:
         if isinstance(other, CheckingAccount):
            return self.acc_number == other.acc_number


    def has_enough_collateral(self, loan: float) -> bool:
        return self.amount >= 2 * loan / 3;

    def get_loan(self, amount : float) -> bool:
        return _has_enough_collateral(float)

    def __str__(self) -> str:
        return f'Checking Account acc number : {self.acc_number}'

    #empty method.
    def add_interest(self) -> None:
        pass
```

Notice that each branch of the original annoying `if else` is now implemented in its class. Now if the client comes back and asks Joe to add a fixed deposit account, Joe will just create a new class called FixedDeposit and it will inherit from that abstract Account class. With this design, note that :

- To add new functionality, we add more classes and ignore all existing classes. This is the **Open Closed Principle.**

Note that the CheckingAccount class leaves the add_interest method empty. This is a code smell known as the **Rebellious Hierarchy** design smell and we shall fix it later when we get to the **Interface Segregation Principle**.

> **REBELLIOUS HIERARCHY** <br>
> This smell arises when a subtype rejects the methods provided by its supertype(s).
> In this smell, a supertype and its subtypes conceptually share an IS-A relationship,
> but some methods defined in subtypes violate this relationship. For example, for
> a method defined by a supertype, its overridden method in the subtype could:
>
> - throw an exception rejecting any calls to the method
> - provide an empty (or NOP i.e., NO Operation) method
> - provide a method definition that just prints “should not implement” message
> - return an error value to the caller indicating that the method is unsupported.

After a year, Joe's client comes back and asks Joe to add a Current account. Guess what Joe does?? You guessed right, he just creates a new class for this new account and inherits from Account class as described below.

**Adding CurrentAccount to the hierarchy**

`Account` remains the parent of `SavingsAccount` and `CheckingAccount`, and gains a third direct subclass,
`CurrentAccount`. The parent still holds `acc_name: String`, `acc_number: String` and `amount: Real` as
private attributes and exposes the same operations listed in the Account class structure above.

All three subclasses inherit `deposit(Real): void`, `get_balance(): Real` and `withdraw(Real): void`.
Each defines public `__eq__(Account): Boolean`, `__str__(): String`, `add_interest(): void` and
`get_loan(Real): Boolean` operations, plus the private `has_enough_collateral(Real): Boolean` helper.
The new current-account behaviour lives in `CurrentAccount`; the existing subclasses do not change.
The class structure does not specify a current account's interest rate or collateral rule.

**[⬆ back to top](#table-of-contents)**

### Use-Descriptive-Names

---

> **You know you are working on clean code when each routine turns out to be pretty much what you expected.**
> — Ward Cunningham

Half the battle of making code readable is choosing good names for the things it does. Everything in
[Naming Things](naming.md#naming-things) applies here, with three additions specific to functions.

**1. A long descriptive name beats a short enigmatic one, and beats a long comment.**

**Bad** :angry:

```python
def calc(d):
    """Calculate the number of days a payment is overdue."""
    ...
```

**Good** :smiley:

```python
def days_overdue(due_date: date) -> int:
    ...
```

The docstring became unnecessary because the signature says it.

**2. Do not fear renaming.** Modern editors rename symbols reliably and your tests will catch what they
miss. Names are cheap to change and expensive to leave wrong.

**3. Be consistent.** Use the same phrases, nouns and verbs across a module so that the API is guessable
(see [Pick One Word per Concept](naming.md#pick-one-word-per-concept)):

```python
find_by_id(...)      find_by_email(...)      find_all(...)
```

A useful trick: **name the function after what it returns, or after the effect it has** — never after how it
does it.

**Bad** :angry:

```python
def loop_through_orders_and_add(orders): ...    # describes the implementation
```

**Good** :smiley:

```python
def total_value(orders: list[Order]) -> Decimal: ...
```

The second name survives the day you replace the loop with a comprehension or a database `SUM`.

**[⬆ back to top](#table-of-contents)**

### Function-Arguments

---

The ideal number of arguments for a function is zero. Next comes one, then two. Three should be avoided
where possible, and more than three needs a very good reason.

Arguments are hard for two reasons: each one is a concept the reader has to hold while reading the name, and
each one multiplies the number of cases a test has to cover.

#### Common forms

**Niladic (zero arguments).** The easiest to understand — but in a function (rather than a method) it usually
means the input arrives through a global or through I/O, which is a side effect. See
[Niladic Functions](#1-niladic-functions).

**Monadic (one argument).** There are two good reasons to pass a single argument: asking a question about it
(`is_overdrawn(account)`), or transforming it into something else (`parse_date(text)`). A third form, an
*event* (`notify_password_changed(user)`), takes an input and changes state without returning anything; use
it deliberately and make the name say so.

**Dyadic (two arguments).** Fine when the two arguments have a natural order or are two halves of one value
— `Point(x, y)`, `assert_equal(expected, actual)`. Awkward when they do not, because the reader has to
remember the order. `write_field(output_stream, name)` is a dyad that would read better as a method:
`output_stream.write_field(name)`.

**Triadic and beyond.** Usually a sign that some of the arguments belong together in an object.

**Bad** :angry:

```python
def create_circle(x: float, y: float, radius: float) -> Circle: ...

create_circle(0, 0, 5)
```

**Good** :smiley:

```python
@dataclass(frozen=True)
class Point:
    x: float
    y: float


def create_circle(center: Point, radius: float) -> Circle: ...

create_circle(Point(0, 0), radius=5)
```

Wrapping `x` and `y` in `Point` is not cheating: those two values genuinely are one concept, and now they
have a name and a place for behaviour like `distance_to`.

#### Flag arguments are ugly

Passing a boolean into a function loudly proclaims that the function does more than one thing: one thing if
the flag is true, another if it is false.

**Bad** :angry:

```python
def render(page: Page, is_suite: bool) -> str:
    if is_suite:
        ...
    else:
        ...
```

**Good** :smiley:

```python
def render_for_suite(page: Page) -> str: ...
def render_for_single_test(page: Page) -> str: ...
```

If you cannot split the function, at least make the flag keyword-only so the call site is readable. Python
gives you this with a bare `*`:

```python
def dump(data: dict, *, indent: bool = False) -> str: ...

dump(payload, indent=True)      # dump(payload, True) is now a TypeError
```

#### Mutable default arguments

This is the classic Python trap, and it is an argument bug rather than a style issue: the default is
evaluated **once**, at function definition time, so every call shares the same list.

**Bad** :angry:

```python
def append_item(item: str, items: list[str] = []) -> list[str]:
    items.append(item)
    return items

append_item('a')    # ['a']
append_item('b')    # ['a', 'b']  <- the same list, still there
```

**Good** :smiley:

```python
def append_item(item: str, items: list[str] | None = None) -> list[str]:
    items = [] if items is None else items
    return [*items, item]
```

#### Prefer keyword arguments at the call site

A call that reads as a sentence needs no comment:

```python
transfer(account_a, account_b, 500)                             # which way round?
transfer(source=account_a, destination=account_b, amount=500)   # obvious
```

#### `*args` and `**kwargs`

Variadic arguments are fine when the function truly treats them uniformly (`print`, `max`, `sum`). They are
a problem when used to paper over an unclear interface, because they erase the signature: neither the reader
nor the type checker can see what the function accepts.

```python
def total(*amounts: Decimal) -> Decimal:        # good: all arguments are the same thing
    return sum(amounts, start=Decimal('0'))


def do_stuff(*args, **kwargs):                  # bad: what does it take? nobody knows
    ...
```

**[⬆ back to top](#table-of-contents)**

### Avoid-Side-Effects

#### **Pure Functions**

---

What on earth is a pure function?? Well, adequately put, a pure function is one without side effects.
Side effects are invisible inputs and outputs from functions. In pure Functional programming,functions behave like mathematical functions. Mathematical functions are transparent-- they will always return the same output when given the same input. Their output only depends on their inputs.

Below are examples of functions with side effects:

#### 1. **Niladic-Functions**

---

**Bad** :angry:

```python
class Customer:
    def __init__(self, first_name : str)-> None:
        self.first_name = first_name

    #This method is impure, it depends on global state.
    def get_name(self):
        return self.first_name

    #more code here
```

Niladic functions have this tendency to depend on some invisible input especially if such a function is member function of a class. Since all class members share the same class variables, most methods aren't pure at all. Class variable values will always depend on which method was called last. In a nutshell, most niladic functions depend on some **global state** in this case `self.first_name`.

The same can be said to functions that return None. These too aren't pure functions. If a function doesn't return, then it is doing something that is affecting global state. **Such functions can not be composed in fluent APIs.** The sort method of the list class has side effects, it changes the list in place whereas the sorted builtin function has not side effects because it returns a new list.

> `sort()` and `reverse()` are now discouraged and instead using the built-in `reversed()` and `sorted()` are encouraged.

**Bad** :angry:

```python
names = ['Kasozi', 'Martin', 'Newton', 'Grady']

#wrong: sorted_names now contains None
sorted_names = names.sort()

#correct: sorted_names now contain the sorted list
sorted_names = sorted(names)
```

> **static methods** <br>
> One way to solve this problem is to use static methods inside a class. Static methods know nothing about the class data and hence their outputs only depend on their inputs.

#### 2. **Argument Mutation**

---

Functions that mutate their input arguments aren't pure functions. This becomes more pronounced when we run on multiple cores. More than one function may be reading from the same variable and each function can be context switched from the CPU at any time. If it was not yet done with editing the variable, others will read garbage.

**Bad** :angry:

```python
from typing import List

Marks = List[int]

marks = [43, 78, 56, 90, 23]

def sort_marks(marks : Marks) -> None:
    marks.sort()

def calculate_average(marks : Marks) -> float:
    return sum(marks)/float(len(marks))
```

From the above code snippet, we have two functions that both read the same list. `sort_marks()` mutates its input argument and this is not good. Now imagine a scenario when `calculate_average_mark()` was running and before it completed, it was context switched and `sort_marks()` allowed to run.

sort_marks will update the list in place and change the order of elements in the list, by the time `calculate_average_average()` will run again, it will be reading garbage.

**Good :smiley:**

```python
from typing import List

Marks = List[int]

marks = [43, 78, 56, 90, 23]

#sort_marks now returns a new list and uses the sorted function

#Mutates input argument
def sort_marks(marks : Marks) -> Marks:
    return sorted(marks)

# Doesn't mutate input argument
def find_average_mark(marks : Marks) -> float:
    return sum(marks)/len(marks)
```

This problem can also be solved by using immutable data structures.

> Function purity is also vital for unit-testing. Impure functions are hard to test especially if the side effect has to do with I/O. Unlike mutation, you can’t avoid side effects related to I/O; whereas mutation is an implementation detail, I/O is usually a requirement.

#### 3. **Exceptions**

---

Some function signatures are more expressive than others, by which I mean that they give us
more information about what the function is doing, what inputs are permissible, and what outputs we can expect. The signature `() → ()`, for example, gives us no information at all: it may print some text, increment a counter, launch a spaceship... who knows! On the other hand, consider this signature:

`(List[int], (int → bool)) → List[int]`

Take a minute and see if you can guess what a function with this signature does. Of course, you
can’t really know for sure without seeing the actual implementation, but you can make an
educated guess. The function returns a list of `ints` as input; it also takes a list of `ints`, as well as a
second argument, which is a function from int to `bool`: a predicate on int.

But is not honest enough. What happens if we pass in an empty list?? This function may throw an exception.

> Exceptions are hidden outputs from functions and functions that use exceptions have side effects.

**Bad** :angry:

```python
def find_quotient(first : int, second : int)-> float:
    try:
        return first/second
    except ZeroDivisionError:
        return None
```

What is wrong with such a function? In its signature, it claims to return a float but we can see that sometimes it fails. Such a function is not honest and such functions should be avoided.

> Functiona languages handle errors using other means like Monads and Options. Not with exceptions.

#### 4. **I/O**

---

Functions that perform input/output aren't pure too. Why? This is because they return different outputs when given the same input argument. Let me explain more about this. Imagine a function that takes in an URL and returns HTML, if the HTML is changed, the function will return a different output but it is still taking in the same URL. Remember mathematical functions don't behave like this.

**Bad** :angry:

```python
def read_HTML(url : str)-> str:
    try:
        with open(url) as file:
            data = file.read()
        data = file.read()
        return data
    except FileNotFoundError:
        print('File Not found')
```

This function is plagued with more than one problem.

- Its signature is not honest. It claims that the function returns a string and takes in a string but from the implementation, we see it can fail.
- This function is performing IO. IO operations produce side effects and thus this function is not pure.

> You can build pure functions in python with the help of the **operator** and **functools** modules. There is a package **fn.py** to support functional programming in Python 2 and 3. According
> to its author, Alexey Kachayev, fn.py provides “implementation of missing features to
> enjoy FP” in Python. It includes a @recur.tco decorator that implements tail-call optimization
> for unlimited recursion in Python, among many other functions, data structures,
> and recipes.

### Command-Query-Separation

---

Functions should either **do** something or **answer** something, but not both. A function that changes the
state of an object is a *command*; a function that reports something about an object is a *query*. Mixing
the two leads to call sites that are impossible to read without opening the function.

**Bad** :angry:

```python
class Account:
    def __init__(self, balance: float) -> None:
        self.balance = balance
        self.is_frozen = False

    def withdraw(self, amount: float) -> bool:
        """Withdraw money and report whether it worked."""
        if self.is_frozen or amount > self.balance:
            return False
        self.balance -= amount
        return True
```

Now read the call site:

```python
if account.withdraw(500):
    ...
```

What does that line mean? "If withdrawing 500 succeeds"? "If we can withdraw 500"? The `if` makes it look
like a question, but money moves as a side effect of asking. Worse, the boolean silently swallows *why* it
failed — a frozen account and insufficient funds are two very different problems for the caller.

**Good** :smiley:

```python
class Account:
    def __init__(self, balance: float) -> None:
        self.balance = balance
        self.is_frozen = False

    def can_withdraw(self, amount: float) -> bool:      # query: no state change
        return not self.is_frozen and amount <= self.balance

    def withdraw(self, amount: float) -> None:          # command: no return value
        if self.is_frozen:
            raise AccountFrozen(self)
        if amount > self.balance:
            raise InsufficientFunds(needed=amount, available=self.balance)
        self.balance -= amount
```

```python
if account.can_withdraw(500):
    account.withdraw(500)
```

Each line now says exactly one thing. Note the second half of the rule at work: **commands report failure by
raising**, not by returning a status code that a caller can forget to check (see
[Exceptions](#3-exceptions)).

#### Recognising a violation

Ask two questions:

- **Can I call this twice in a row and get the same answer?** If not, it is not a pure query.
- **Does an `if` statement around this call change the state of the program?** If yes, you have a
  command-query hybrid.

#### When to break the rule

Sometimes the atomic version is the correct one, because splitting it opens a race window:

```python
if account.can_withdraw(500):     # another thread withdraws here...
    account.withdraw(500)         # ...and this raises
```

For concurrent code, a single atomic operation is right — but then name it as a command and let it raise, or
return a *result object* that describes what happened, rather than a bare boolean:

```python
@dataclass(frozen=True)
class WithdrawalResult:
    succeeded: bool
    new_balance: Decimal
    reason: str | None = None
```

Python's own library shows both styles: `dict.get(key)` is a query, `dict.pop(key)` is deliberately a hybrid
because atomicity matters, and its name (`pop`, not `get`) warns you that it mutates.

**[⬆ back to top](#table-of-contents)**

### Don't Repeat Yourself (DRY)

---

Let us imagine that we are working on a banking application. We all know that such an application will manipulate bank account objects among other things.
Let us assume that at the start of the project, we have only two types of accounts to work with;

- Savings Account
- Checking Account

We roll up our sleeves and put our OOP knowledge to test. We craft two classes to model both and Savings and Checking accounts.

**SavingsAccount class**

**Bad** :angry:

```python
class SavingsAccount:
    def __init__(self, acc_number: str, amount: float, name: str):
        self.acc_number = acc_number
        self.amount = amount
        self.name = name

    def get_balance(self) -> float:
        return self.amount

    def __eq__(self, other: SavingsAccount) -> bool:
        if isinstance(other, SavingsAccount):
            return self.acc_number == other.acc_number

    def deposit(self, amount: float) -> bool:
        if amount > 0:
            self.amount += amount

    def withdraw(self, amount: float) -> None:
        if (amount > 0) and (amount <= self.amount):
            self.amount -= amount

    def has_enough_collateral(self, loan: float) -> bool:
        if loan < self.amount / 2:
            return True

    def __str__(self) -> str:
        return f'Saving Account acc number : {self.acc_number}'

    def add_interest(self) -> None:
        self.deposit(0.1 * self.amount)
```

**CheckingAccount class**

**Bad** :angry:

```python
class CheckingAccount:
    def __init__(self, acc_number: str, amount: float, name: str):
        self.acc_number = acc_number
        self.amount = amount
        self.name = name

    def get_balance(self) -> float:
        return self.amount

    def __eq__(self, other: SavingsAccount) -> bool:
        if isinstance(other, SavingsAccount):
            return self.acc_number == other.acc_number

    def deposit(self, amount: float) -> bool:
        if amount > 0:
            self.amount += amount

    def withdraw(self, amount: float) -> None:
        if (amount > 0) and (amount <= self.amount):
            self.amount -= amount

    def has_enough_collateral(self, loan: float) -> bool:
        if loan < self.amount / 5:
            return True

    def __str__(self) -> str:
        return f'Checking Account acc number : {self.acc_number}'

    def add_interest(self) -> None:
        self.deposit(0.5 * self.amount)
```

The table describes all the methods added to both classes.

| method                    | description                                         |
| ------------------------- | --------------------------------------------------- |
| `get_balance()`           | returns the account balance                         |
| `__str__()`               | returns the string representation of account object |
| `add_interest()`          | adds a given interest to a given account            |
| `has_enough_collateral()` | checks if the account can be granted a loan         |
| `withdraw()`              | withdraws a given amount from the account           |
| `deposit()`               | deposits an amount to the account                   |
| `__eq__()`                | checks if 2 accounts are the same                   |

The class structures of both classes are shown below. Notice the duplication in method names.

| Member | Visibility | `SavingsAccount` | `CheckingAccount` |
| --- | --- | --- | --- |
| `acc_number` | Private | `String` | `String` |
| `amount` | Private | `Real` | `Real` |
| `name` | Private | `String` | `String` |
| `__eq__` | Public | `__eq__(SavingsAccount): Boolean` | `__eq__(CheckingAccount): Boolean` |
| `__str__` | Public | `__str__(): String` | `__str__(): String` |
| `add_interest` | Public | `add_interest(): void` | `add_interest(): void` |
| `deposit` | Public | `deposit(Real): void` | `deposit(Real): void` |
| `get_balance` | Public | `get_balance(): Real` | `get_balance(): Real` |
| `has_enough_collateral` | Public | `has_enough_collateral(Real): Boolean` | `has_enough_collateral(Real): Boolean` |
| `withdraw` | Public | `withdraw(Real): Real` | `withdraw(Real): void` |

These are two separate classes, with no shared parent in this design. The original UML gives the savings
withdrawal a `Real` return type, although the Python implementation returns no value.

If you look more closely, both these classes contain the same methods and to make it worse, most of these methods contain exactly the same code. This is a **bad** practice and it leads to a maintenance nightmare. Identical code is littered in more than one place and so if we ever make changes to one of the copies, we have to change all the others.

There is a software principle that helps in solving such a problem and this principle is known as **DRY** for Don't Repeat Yourself.

> The “Don’t Repeat Yourself” Rule <br>
> A piece of code should exist in exactly one place.

It is evident from our bad design that we have two classes that both claim to do same thing really well and so we just violated the **Most Qualified Rule**. In most cases, such scenarios arise due to failing to identify similarities between objects in a system.

To solve this problem, we will use inheritance. We will define a new abstract class called BankAccount and we will implement all the method containing the similar logic in this abstract class. Then we will leave the different methods to be implemented by subclasses of BankAccount.

Below is the class structure for our new design.

**BankAccount inheritance hierarchy**

`SavingsAccount` and `CheckingAccount` both inherit from `BankAccount`. The parent owns the protected
attributes `acc_number: String`, `amount: Real` and `name: String`.

| Public operation on `BankAccount` | Implementation |
| --- | --- |
| `__eq__(BankAccount): Boolean` | Each subclass supplies its equality behaviour. |
| `__str__(): String` | Each subclass supplies its string representation. |
| `add_interest(): void` | Each subclass supplies its interest behaviour. |
| `deposit(Real): void` | Shared implementation in `BankAccount`. |
| `get_balance(): Real` | Shared implementation in `BankAccount`. |
| `has_enough_collateral(Real): Boolean` | Each subclass supplies its collateral rule. |
| `withdraw(Real): void` | Shared implementation in `BankAccount`. |

Both subclasses declare `__eq__(BankAccount): Boolean`, `__str__(): String`, `add_interest(): void` and
`has_enough_collateral(Real): Boolean` as public operations. The shared state and the other three methods
are inherited instead of duplicated.

**BankAccount** class

**Good** :smiley:

```python
from abc import ABC, abstractmethod

class BankAccount(ABC):
    def __init__(self, acc_number: str, amount: float, name: str):
        self.acc_number = acc_number
        self.amount = amount
        self.name = name

    def get_balance(self) -> float:
        return self.amount

    def deposit(self, amount: float) -> bool:
        if amount > 0:
            self.amount += amount

    def withdraw(self, amount: float) -> None:
        if (amount > 0) and (amount <= self.amount):
            self.amount -= amount

    @abstractmethod
    def __eq__(self, other: SavingsAccount) -> bool:
        pass

    @abstractmethod
    def has_enough_collateral(self, loan: float) -> bool:
        pass

    @abstractmethod
    def __str__(self) -> str:
        pass

    @abstractmethod
    def add_interest(self) -> None:
        pass
```

> **Note :** In the BankAccount abstract class, the methods `__eq__()`, `has_enough_collateral()`, `__str__()` and `add_interest()` are abstract and so it is the responsible of subclasses to implement them.

**SavingsAccount class**

**Good** :smiley:

```python
class SavingAccount(BankAccount):
    def __init__(self, acc_number: str, amount: float, name: str):
        BankAccount.__init__(self, acc_number, amount, name)

    def __eq__(self, other: SavingsAccount) -> bool:
         if isinstance(other, SavingsAccount):
            return self.acc_number == other.acc_number


    def has_enough_collateral(self, loan: float) -> bool:
        if loan < self.amount / 2:
            return True


    def __str__(self) -> str:
        return f'Saving Account acc number : {self.acc_number}'


    def add_interest(self) -> None:
        self.deposit(0.1 * self.amount)
```

**CheckingAccount class**

**Good** :smiley:

```python
class CheckingAccount(BankAccount):
    def __init__(self, acc_number: str, amount: float, name: str):
        BankAccount.__init__(self, acc_number, amount, name)

    def __eq__(self, other: SavingsAccount) -> bool:
         if isinstance(other, CheckingAccount):
            return self.acc_number == other.acc_number


    def has_enough_collateral(self, loan: float) -> bool:
        if loan < self.amount / 5:
            return True


    def __str__(self) -> str:
        return f'Checking Account acc number : {self.acc_number}'


    def add_interest(self) -> None:
        self.deposit(0.5 * self.amount)
```

With this new design, if we ever want to modify the methods common to both classes, we only edit them in the abstract class. This simplifies our codebase maintenance. In fact, this was of organizing code is so ideal for implementing the **Replace Ifs with Polymorphism (RIP)** principle as we shall see later.

