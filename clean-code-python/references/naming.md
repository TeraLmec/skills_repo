## Naming Things

## Table of Contents

- [Naming Things](#naming-things)
- [Use intention revealing names](#use-intention-revealing-names)
- [Meaningful-Distinctions](#meaningful-distinctions)
- [Avoid-Disinformation](#avoid-disinformation)
- [Do not lie about the type](#do-not-lie-about-the-type)
- [Do not lie about the meaning](#do-not-lie-about-the-meaning)
- [Do not use names with a foreign meaning](#do-not-use-names-with-a-foreign-meaning)
- [Beware of near identical names](#beware-of-near-identical-names)
- [Never use lower case l or upper case O as names](#never-use-lower-case-l-or-upper-case-o-as-names)
- [Pronounceable-names](#pronounceable-names)
- [When abbreviations are fine](#when-abbreviations-are-fine)
- [Searchable-Names](#searchable-names)
- [Don't-be-cute](#dont-be-cute)
- [Avoid-Encodings](#avoid-encodings)
- [Hungarian-Notation](#hungarian-notation)
- [Member-Prefixes](#member-prefixes)
- [Interfaces-&-Implementations](#interfaces--implementations)
- [Gratuitous-Context](#gratuitous-context)
- [Avoid-Mental-Mapping](#avoid-mental-mapping)
- [Class-Names](#class-names)
- [Types-of-Objects](#types-of-objects)
- [Simple-Superclass-Name](#simple-superclass-name)
- [Qualified-Subclass-Name](#qualified-subclass-name)
- [Method-Names](#method-names)
- [Pick-One-Word-per-Concept](#pick-one-word-per-concept)
- [Don't-Pun](#dont-pun)
- [Use-Solution-Domain-Names](#use-solution-domain-names)
- [Use-Problem-Domain-Names](#use-problem-domain-names)
- [Add-Meaningful-Context](#add-meaningful-context)


---

Modern software is so complex that no one can understand all parts of a non-trivial project alone. The only way humans tame details is through abstractions. With abstraction, we focus on the essential and forget about the non-essential at that particular time. You remember the way you learned body biology?? You focused on one system at a time, digestive, nervous, cardiovascular e.t.c and ignored the rest. That is abstraction at work.

A variable name is an abstraction over memory, a function name is an abstraction over logic, a class name is an abstraction over a packet of data and the logic that operates on that data.

The most fundamental abstraction in writing software is **naming**. Naming things is just one part of the story, using good names is a skill that unfortunately, is not owned by most programmers and that is why we have come up with so many refactorings concerned with naming things.

Good names bring order to the chaotic environment of crafting software and hence, we better be good at this skill so that we can enjoy our craft.

**[⬆ back to top](#table-of-contents)**

### **Use intention revealing names**

---

This rule enforces that programmers should make their code read like well written prose by naming parts <br>
of their code perfectly. With such good naming, a programmer will never need to resort to comments or unnecessary <br> doc strings.
Below is a code snippet from a software system. Would you make sense of it without any explanation?

**Bad** :angry:

```python
from typing import List

def f(a : List[List[int]])->List[List[int]]:
    return [i for i in a if i[1] == 0]
```

It would be ashaming that someone would fail to understand such a simple function. What could have gone wrong??

The problem is simple.This code is littered with **mysterious names**. We have to agree that this is code and not a detective novel. Code should be clear and precise.

What this code does is so trivial. It takes in a collection of orders and returns the pending orders. Let's pause for a moment and appreciate the extreme over engineering in this solution.
The programmer assumes that each order is coded as a list of `ints` (`List[int]`) and that the second element is the order status. He decides that 0 means pending and 1 means cleared.

Notice the first problem... that snippet doesn't contain knowledge about the domain. This is a design smell known as a **missing abstraction**. We are missing the Order abstraction.

> **Missing Abstraction** <br>
> This smell arises when clumps of data are used instead creating a class or an interface

We have a thorny problem right now, we lack meaningful domain abstractions. One of the ways of solving the missing abstraction smell is to **map domain entities**. So lets create an abstraction called Order.

```python
from typing import List

class Order:
    def __init__(self, order_id : int, order_status : int) -> None:
        self._order_id = order_id
        self._order_status = order_status

    def is_pending(self) -> bool:
        return self._order_status == 0

    #more code goes here
```

> We could also have used the **namedtuple** in the python standard library but we won't be too functional that early. Let us stick with OOP for now. **namedtuples** contain only data and not data with code that acts on it.

Let us now refactor our entity names and use our newly created abstraction too. We arrive at the following code snippet.

**Better: :smiley:**

```python
from typing import List

Orders = List[Order]

def get_pending_orders(orders : Orders)-> Orders:
    return [order for order in orders if order.is_pending()]
```

This function reads like well written prose.

Notice that the `get_pending_orders()` function delegates the logic of finding the order status to the Order class. This is because the Order class knows its internal representation more than anyone else, so it better implement this logic. This is known as the **Most Qualified Rule** in OOP.

> **Most Qualified Rule** <br>
> Work should be assigned to the class that knows best how to do it.

> We are using the listcomp for a reason. Listcomps are examples of **iterative expressions**. They serve one role and that is creating lists. On the other hand, for-loops are **iterative commands** and thus accomplish a myriad of tasks. Pick the right tool for the job.

Never allow client code know your implementation details. In fact the ACM A.M Laureate Babra Liskov says it soundly in her book [Program development in Java. Abstraction, Specification and OOD](https://book4you.org/book/1164544/93467d). The **Iterator design pattern** is one way of solving that problem.

Here is another example of a misleading variable name.

**Bad** :angry:

```python
student_list= {'kasozi','vincent', 'bob'}
```

This variable name is so misleading.

- It contains noise. why the list suffix?
- It is lying to us. Lists are not the same as sets. They may all be collections but they are not the same at all.

To prove that lists are not sets, below is a code snippet that returns the methods in the List class that aren't in the Set class.

```python
sorted(set(dir(list())) - set(dir(set())))
```

Once it has executed, `append()` is one of the returned functions implying that sets don't support `append()` but instead support `add()`. So you write the code below, your code breaks.

> Sets are not sequences like lists. In fact, they are unordered collections and so adding the `append()` method to the set class would be misleading. `append()` means we are adding at the end which may not be the case with sets.

**Bad** :angry:

```python
student_list= {'kasozi','vincent', 'bob'}
student_list.append('martin') #It breaks!!
```

It is better to use a different variable name is neutral to the data structure being used.
In this case, once you decide to change data structure used, your variable won't destroy the semantics of your code.

**Good** :smiley:

```python
students = {'kasozi', 'vincent', 'bob'}
```

> You can not achieve good naming with a bad design. You can see that mapping domain entities into our code has made our codebase use natural names.

**[⬆ back to top](#table-of-contents)**

### Meaningful-Distinctions

---

When two things in your code are different, their names must tell you **how** they are different. Names that
differ only by a noise word, a number or a synonym force the reader to open both definitions and diff them
by hand.

Number series names (`a1`, `a2`, ... `aN`) are the laziest form of this. They carry no information at all.

**Bad** :angry:

```python
from typing import List

def copy_marks(source: List[int], destination: List[int]) -> None:
    for i in range(len(source)):
        destination[i] = source[i]
```

Compare that with the version the reader actually has to decode:

**Bad** :angry:

```python
from typing import List

def copy(a1: List[int], a2: List[int]) -> None:
    for i in range(len(a1)):
        a2[i] = a1[i]
```

Is `a1` the source or the destination? You cannot know without reading the body. `source` and `destination`
answer the question in the signature.

**Noise words** are the subtler version of the same mistake. `Info`, `Data`, `Object`, `Manager`, `Variable`
and `The` are all words that could be deleted without changing the meaning of the name.

**Bad** :angry:

```python
class Customer:
    ...

class CustomerData:      # how is this different from Customer?
    ...

class CustomerInfo:      # ...or from this?
    ...

class CustomerObject:    # ...or this?
    ...
```

If you cannot explain to a colleague when to reach for `Customer` and when to reach for `CustomerData`, then
you do not have four concepts, you have one concept and three redundant classes.

**Good** :smiley:

```python
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)
class Customer:
    """A customer as the business talks about them."""
    customer_id: str
    full_name: str


@dataclass(frozen=True)
class CustomerCreditProfile:
    """Everything the credit department needs to score a customer."""
    customer_id: str
    credit_limit: Decimal
    outstanding_balance: Decimal
```

Now the two names describe two genuinely different ideas, and the distinction lives in the name rather than
in tribal knowledge.

The same rule applies to functions. If a module exposes `get_account()`, `fetch_account()` and
`retrieve_account()`, a reader has to assume there are three different behaviours, because otherwise why
would there be three names? Pick one verb per concept and stick to it (see
[Pick One Word per Concept](#pick-one-word-per-concept)).

**Bad** :angry:

```python
def get_active_users(): ...
def fetch_active_accounts(): ...   # same thing, different words
def retrieve_live_customers(): ... # same thing again
```

**Good** :smiley:

```python
def get_active_users(): ...
def get_active_accounts(): ...
def get_active_customers(): ...
```

> **Rule of thumb:** if you can swap two names in your head and the code still makes sense, the names are not
> meaningfully distinct.

**[⬆ back to top](#table-of-contents)**

### Avoid-Disinformation

---

A name that is merely vague slows the reader down. A name that is **wrong** sends them in the opposite
direction, and they will trust it, because names are the only documentation people actually read.

#### Do not lie about the type

Suffixes such as `list`, `dict` or `str` are a promise. Break the promise and you have planted a bug in the
reader's mind.

**Bad** :angry:

```python
account_list = {'123': 400.0, '456': 250.0}   # it is a dict, not a list
```

**Good** :smiley:

```python
balance_by_account = {'123': 400.0, '456': 250.0}
```

`balance_by_account['123']` reads exactly like what it does, and it survives the day someone swaps the dict
for an ordered mapping or a database row.

#### Do not lie about the meaning

**Bad** :angry:

```python
def is_valid(email: str) -> bool:
    return '@' in email and email.endswith('.com')
```

The name claims general validity; the body only accepts `.com` addresses. Every caller that trusts the name
ships a bug for `.org` users.

**Good** :smiley:

```python
def is_dotcom_email(email: str) -> bool:
    return '@' in email and email.endswith('.com')
```

The behaviour did not change, but nobody is misled any more, and the awkward name makes the arbitrary rule
visible so it can be questioned.

#### Do not use names with a foreign meaning

`hp`, `aix` and `sco` are names of Unix platforms. `Account` in a banking system means something specific to
the domain experts. Reusing such a word for something else is disinformation even if the spelling is
innocent.

#### Beware of near identical names

```python
XYZControllerForEfficientHandlingOfStrings
XYZControllerForEfficientStorageOfStrings
```

These differ by two words in the middle of a thirty-character name. Autocomplete will happily hand you the
wrong one, and code review will not catch it.

#### Never use lower case `l` or upper case `O` as names

**Bad** :angry:

```python
l = 1
O = 0
if O == l:      # is that a zero and a one, or two letters?
    l = O
```

In most fonts `l` is indistinguishable from `1` and `O` from `0`.

**Good** :smiley:

```python
line_count = 1
offset = 0
if offset == line_count:
    line_count = offset
```

> **Type hints are part of the name.** `def process(items)` tells you nothing;
> `def refund(orders: list[Order]) -> list[Refund]` cannot lie without the type checker noticing. Let
> `mypy` or `pyright` keep your names honest.

**[⬆ back to top](#table-of-contents)**

### Pronounceable-names

---

When naming things in your code, it is much better to use names that are easy to pronounce by programmers.
This enables developers to discuss the code without the need to sound silly as they mention the names. If
you are a polyglot in natural languages, it is much better to use the language common to most developers
when naming your entities.

Humans have evolved for language. That part of your brain is doing real work while you read code, so a name
you can say out loud is a name you can hold in your head, remember tomorrow and mention in a stand-up
without spelling it letter by letter.

**Bad** :angry:

```python
from typing import List
import math

def sqrs(first_n: int) -> List[int]:
    if first_n > 0:
        return [int(math.pow(i, 2)) for i in range(first_n)]
    return []

lstsqrs = sqrs(5)
```

How can a human pronounce `sqrs` and `lstsqrs`? This is a serious problem. Let's correct it.

**Good** :smiley:

```python
from typing import List
import math

def generate_squares(first_n: int) -> List[int]:
    if first_n > 0:
        return [int(math.pow(i, 2)) for i in range(first_n)]
    return []

squares = generate_squares(5)
```

The problem shows up most painfully in data records, where the unpronounceable name is repeated at every
call site:

**Bad** :angry:

```python
from dataclasses import dataclass

@dataclass
class DtaRcrd102:
    genymdhms: str      # "gen why emm dee aich emm ess"?
    modymdhms: str
    pszqint: str = '102'
```

**Good** :smiley:

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Customer:
    generation_timestamp: datetime
    modification_timestamp: datetime
    record_id: str = '102'
```

The second version supports an actual conversation: *"Hey Bob, look at the record with generation timestamp
of last Tuesday."*

#### When abbreviations are fine

Do not overcorrect. An abbreviation that every programmer in the room already pronounces as a word is a real
word for naming purposes:

```python
html_response          # not hyper_text_markup_language_response
http_client            # not hyper_text_transfer_protocol_client
url, uuid, csv, json, api, db, id
```

The test is social, not textual: **can two developers say this name out loud to each other and both know
what it means?** If yes, keep it. If it comes out as a spelling bee, rename it.

Loop counters are the traditional exception. `i`, `j` and `k` inside a three-line loop are pronounceable and
carry decades of shared convention, so they are fine; `i` as a module-level variable is not.

**[⬆ back to top](#table-of-contents)**

### Searchable-Names

---

For example, you are looking for some part of the code where you calculate something and you remember that
it was about work days in a week.

**Bad** :angry:

```python
for i in range(0, 34):
    s += (t[i] * 4) / 5
```

What is easier to find, `5` or `WORK_DAYS_PER_WEEK`? Searching for `5` in any real codebase returns
thousands of hits, and none of them tell you whether they mean the same five.

Single letter names and bare numeric literals ("magic numbers") share the same flaw: they cannot be found,
and they cannot be changed with confidence. If the working week ever becomes four days, the `5` above is
indistinguishable from every other `5` in the system.

It is normal to name a local variable with one character in a short function, but if you can avoid it, do.

**Good** :smiley:

```python
from typing import Final, List

REAL_DAYS_PER_IDEAL_DAY: Final[int] = 4
WORK_DAYS_PER_WEEK: Final[int] = 5


def estimate_weeks(task_estimates: List[int]) -> float:
    total_weeks = 0.0
    for task_estimate in task_estimates:
        real_task_days = task_estimate * REAL_DAYS_PER_IDEAL_DAY
        total_weeks += real_task_days / WORK_DAYS_PER_WEEK
    return total_weeks
```

Three things improved at once:

1. Every constant is now greppable, and `Final` tells both the reader and the type checker that it is not
   meant to be reassigned.
2. The loop iterates over the collection instead of over a hard coded `range(0, 34)` that silently breaks
   when the number of tasks changes.
3. `sum` is no longer shadowed. Naming a variable `sum`, `list`, `id`, `type` or `input` hides a builtin and
   is a bug waiting for the line that needs the real one.

> The length of a name should be proportional to the size of its scope. A three-line comprehension can use
> `n`; a module level constant that appears in twenty files deserves `WORK_DAYS_PER_WEEK`.

**[⬆ back to top](#table-of-contents)**

### Don't-be-cute

---

Humour ages badly and does not survive translation. A joke name is only funny to the person who wrote it, on
the day they wrote it, in their culture, and it is a puzzle for everybody else forever after.

**Bad** :angry:

```python
def whack()   -> None: ...   # deletes a record
def eat_my_shorts() -> None: ...   # aborts the job
def holy_hand_grenade() -> None: ...   # clears the cache
```

**Good** :smiley:

```python
def delete_record() -> None: ...
def abort_job() -> None: ...
def clear_cache() -> None: ...
```

The same applies to slang and colloquialisms. `kill_it()`, `blow_away()` and `nuke()` all mean "delete", but
only `delete()` means it in every English speaking country and in every translation of your documentation.

> **Say what you mean. Mean what you say.** Cleverness in a name is a cost paid by every future reader so
> that one author could enjoy a moment.

**[⬆ back to top](#table-of-contents)**

### Avoid-Encodings

---

Encoding type or scope information into a name was invented for languages and editors that could not tell
you either. Python has type hints, and your editor has "go to definition"; the encoding is now pure
overhead that has to be maintained by hand and that silently rots the moment the type changes.

**Bad** :angry:

```python
str_name = 'kasozi'
i_count = 3
f_rate = 0.5
l_accounts = ['a', 'b']
```

**Good** :smiley:

```python
name: str = 'kasozi'
count: int = 3
rate: float = 0.5
accounts: list[str] = ['a', 'b']
```

The annotation says the same thing, a type checker verifies it, and renaming a type does not leave a lie
behind in the identifier.

**[⬆ back to top](#table-of-contents)**

#### Hungarian-Notation

---

Hungarian Notation prefixes each name with a code for its type: `strName`, `iCount`, `bIsReady`,
`arrItems`. It made sense in 1980s C, where a compiler was weakly typed and an editor could not tell you
anything about a symbol.

**Bad** :angry:

```python
def calculate(fPrice: float, iQuantity: int) -> float:
    fTotal = fPrice * iQuantity
    return fTotal
```

**Good** :smiley:

```python
def calculate_total(price: float, quantity: int) -> float:
    return price * quantity
```

The real damage is that the prefix eventually lies. The day `iQuantity` becomes a `Decimal`, you either
rename it in every file or you leave a permanent piece of disinformation in the codebase. Type hints move
with the code because tooling checks them.

**[⬆ back to top](#table-of-contents)**

#### Member-Prefixes

---

You do not need to prefix instance attributes to mark them as members. Inside a class, `self.` already says
it, and outside a class the attribute is reached through an object.

**Bad** :angry:

```python
class Account:
    def __init__(self, balance: float) -> None:
        self.m_balance = balance          # 'm_' for member
        self.m_owner_name = 'unknown'

    def deposit(self, amount: float) -> None:
        self.m_balance += amount
```

**Good** :smiley:

```python
class Account:
    def __init__(self, balance: float) -> None:
        self.balance = balance
        self.owner_name = 'unknown'

    def deposit(self, amount: float) -> None:
        self.balance += amount
```

Python does have one meaningful naming convention here, and it is about **visibility, not membership**:

```python
class Account:
    def __init__(self, balance: float) -> None:
        self.balance = balance      # public API
        self._ledger = []           # internal, subject to change without notice
        self.__token = 'secret'     # name mangled to _Account__token
```

- A single leading underscore is a convention meaning *"this is not part of the public interface"*. Nothing
  enforces it; it is a message to humans and to linters.
- A double leading underscore triggers name mangling. Use it only when you genuinely need to avoid a name
  clash in a subclass, not as a security feature.

If a class has so many attributes that you feel the urge to prefix them for organisation, the class is
probably doing too much. Split it (see the
[Single Responsibility Principle](single-responsibility.md#single-responsibility-principle)).

**[⬆ back to top](#table-of-contents)**

#### Interfaces-&-Implementations

---

In some ecosystems the convention is to mark the interface: `IShapeFactory` and `ShapeFactory`. If you have
to encode one of the two, prefer encoding the implementation, because the interface is the name that callers
depend on and it should be the clean one.

Python's version of this problem shows up with `abc.ABC` and `typing.Protocol`.

**Bad** :angry:

```python
from abc import ABC, abstractmethod

class IPaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount: float) -> None: ...

class PaymentGateway(IPaymentGateway):
    def charge(self, amount: float) -> None: ...
```

The caller now depends on a name with a stray `I` in it, and the two names are one character apart.

**Good** :smiley:

```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    """What every gateway must be able to do."""

    @abstractmethod
    def charge(self, amount: float) -> None: ...


class StripeGateway(PaymentGateway):
    def charge(self, amount: float) -> None: ...


class FakeGateway(PaymentGateway):
    """Used by the test-suite; records charges instead of making them."""

    def __init__(self) -> None:
        self.charges: list[float] = []

    def charge(self, amount: float) -> None:
        self.charges.append(amount)
```

The abstraction owns the plain name; each implementation is named after what makes it different.

Often you do not need the base class at all. A `Protocol` gives you the same static guarantees with no
inheritance and no runtime coupling:

```python
from typing import Protocol

class PaymentGateway(Protocol):
    def charge(self, amount: float) -> None: ...


def check_out(total: float, gateway: PaymentGateway) -> None:
    gateway.charge(total)
```

Any object with a matching `charge` method satisfies `PaymentGateway`, including one written by a library you
do not control.

**[⬆ back to top](#table-of-contents)**

### Gratuitous-Context

---

Adding context is good; adding the *same* context to every name in the system is noise. If you are building
the "Gas Station Deluxe" application, do not prefix every class with `GSD`.

**Bad** :angry:

```python
# file: gas_station/gsd_models.py
class GSDAccount: ...
class GSDAccountAddress: ...
class GSDCustomer: ...

def gsd_calculate_gsd_account_total(gsd_account: GSDAccount) -> float: ...
```

Typing `GSD` into autocomplete now offers every class in the application, which is exactly as useful as no
autocomplete at all.

**Good** :smiley:

```python
# file: gas_station/models.py
class Account: ...
class Address: ...
class Customer: ...

def calculate_total(account: Account) -> float: ...
```

The module already supplies the context, and Python lets the caller decide how much of it to keep:

```python
from gas_station import models

account = models.Account()
```

The same applies inside a class. A method on `Account` does not need to repeat the class name:

**Bad** :angry:

```python
class Account:
    def get_account_balance(self) -> float: ...
    def set_account_owner(self, owner: str) -> None: ...
```

**Good** :smiley:

```python
class Account:
    def get_balance(self) -> float: ...
    def set_owner(self, owner: str) -> None: ...
```

`account.get_balance()` already reads as a sentence; `account.get_account_balance()` stutters.

> Shorter names are generally better than longer ones, **so long as they are clear**. Add no more context
> to a name than is necessary.

**[⬆ back to top](#table-of-contents)**

### Avoid-Mental-Mapping

---

Readers should not have to mentally translate your names into names they already know. This is the problem
with single letter variables outside a tiny scope: the reader has to keep a private lookup table in their
head while they read, and every lookup is a chance to get it wrong.

**Bad** :angry:

```python
def process(d: dict) -> list:
    r = []
    for k, v in d.items():
        if v > 0:
            t = k.upper()
            r.append((t, v))
    return r
```

To review that function you must first decide that `d` is a stock level per product, `r` is the result, `k`
is a product code, `v` is a quantity and `t` is the normalised code. None of it is written down.

**Good** :smiley:

```python
def in_stock_products(stock_by_product: dict[str, int]) -> list[tuple[str, int]]:
    available = []
    for product_code, quantity in stock_by_product.items():
        if quantity > 0:
            available.append((product_code.upper(), quantity))
    return available
```

Nothing has to be decoded; the names are the explanation. And once the names are honest, the simplification
becomes obvious:

```python
def in_stock_products(stock_by_product: dict[str, int]) -> list[tuple[str, int]]:
    return [
        (product_code.upper(), quantity)
        for product_code, quantity in stock_by_product.items()
        if quantity > 0
    ]
```

> **Professionals write code that others can understand.** One difference between a smart programmer and a
> professional programmer is that the professional understands that clarity is king.

**[⬆ back to top](#table-of-contents)**

### Class-Names

---

A class is a noun. It models a thing, so its name should be a noun or a noun phrase in `PascalCase`:
`Customer`, `WikiPage`, `Account`, `AddressParser`.

Avoid verbs, and be suspicious of the words `Manager`, `Processor`, `Data`, `Info`, `Handler` and `Util`.
They are not wrong by definition, but they are the words we reach for when we do not know what a class is
for, and a class that we cannot name is usually a class that does too much.

**Bad** :angry:

```python
class DataManager:
    def handle(self, stuff): ...
```

**Good** :smiley:

```python
class InvoiceRepository:
    def save(self, invoice: 'Invoice') -> None: ...
    def find_by_id(self, invoice_id: str) -> 'Invoice': ...
```

`InvoiceRepository` tells you what it holds, where it sits in the design and what you may ask of it.

**[⬆ back to top](#table-of-contents)**

#### Types-of-Objects

---

Most classes fall into one of a handful of roles, and naming is much easier once you know which role you are
naming. A useful split:

| Role | What it is | Naming pattern | Example |
| --- | --- | --- | --- |
| **Value object** | Immutable, compared by value, no identity | The concept itself | `Money`, `EmailAddress`, `DateRange` |
| **Entity** | Has an identity that outlives its attributes | The domain noun | `Customer`, `Order`, `Account` |
| **Service** | Stateless behaviour over other objects | Verb phrase turned noun | `PaymentProcessor`, `TaxCalculator` |
| **Repository / Gateway** | Talks to storage or to the outside world | `<Noun>Repository`, `<Noun>Gateway` | `OrderRepository`, `StripeGateway` |
| **Factory / Builder** | Constructs other objects | `<Noun>Factory`, `<Noun>Builder` | `ReportFactory`, `QueryBuilder` |
| **Data transfer object** | A bag of fields crossing a boundary | `<Noun>Request` / `<Noun>Response` | `CreateOrderRequest` |

```python
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)      # value object: two Money objects with the same
class Money:                 # amount and currency ARE the same money
    amount: Decimal
    currency: str


@dataclass                   # entity: two customers with the same name are
class Customer:              # still two different customers
    customer_id: str
    name: str
```

Mixing roles in one class is the most common source of unmaintainable code. A `Customer` that also knows how
to save itself to Postgres is an entity and a repository at once, and it now has two reasons to change.

**[⬆ back to top](#table-of-contents)**

#### Simple-Superclass-Name

---

The higher a class sits in a hierarchy, the more abstract it is, and the shorter and more general its name
should be. A base class name is a promise about the whole family, so it must not describe any one member.

**Bad** :angry:

```python
class AbstractBaseSavingsAccountImplementation: ...
class CheckingAccount(AbstractBaseSavingsAccountImplementation): ...
```

The base name mentions savings, so a checking account inheriting from it reads as nonsense, and the
`Abstract`, `Base` and `Implementation` noise adds nothing.

**Good** :smiley:

```python
from abc import ABC, abstractmethod


class Account(ABC):
    @abstractmethod
    def add_interest(self) -> None: ...


class SavingsAccount(Account): ...
class CheckingAccount(Account): ...
```

> A short, general superclass name is a sign that the abstraction is real. If you struggle to name the base
> class without listing its subclasses, the hierarchy is probably wrong.

**[⬆ back to top](#table-of-contents)**

#### Qualified-Subclass-Name

---

Subclass names are the mirror image: they carry the qualifier that says **how this one differs** from the
family. The convention is `<Qualifier><Superclass>`.

**Good** :smiley:

```python
class Account(ABC): ...

class SavingsAccount(Account): ...
class CheckingAccount(Account): ...
class FixedDepositAccount(Account): ...
```

Read as a sentence, `SavingsAccount is an Account` is true, which is a cheap first test of the
[Liskov Substitution Principle](liskov-substitution.md#liskov-substitution-principle).

Watch for qualifiers that describe an implementation detail rather than a kind:

**Bad** :angry:

```python
class FastAccount(Account): ...      # fast is not a kind of account
class Account2(Account): ...         # a version number is not a qualifier
class AccountImpl(Account): ...      # 'Impl' says nothing
```

**Good** :smiley:

```python
class CachedAccountRepository(AccountRepository): ...   # caching IS the distinction
class PostgresAccountRepository(AccountRepository): ... # so is the backing store
```

**[⬆ back to top](#table-of-contents)**

### Method-Names

---

If a class is a noun, a method is a verb. Methods do things, so name them with a verb or a verb phrase in
`snake_case`: `save`, `delete_page`, `calculate_interest`.

**Bad** :angry:

```python
class Account:
    def balance_calculation(self) -> float: ...   # noun phrase
    def new_owner(self, owner: str) -> None: ...  # ambiguous: get or set?
```

**Good** :smiley:

```python
class Account:
    def calculate_balance(self) -> float: ...
    def change_owner(self, owner: str) -> None: ...
```

Predicates that return a `bool` read best as questions: `is_`, `has_`, `can_`, `should_`.

```python
def is_overdrawn(self) -> bool: ...
def has_enough_collateral(self, loan: float) -> bool: ...
def can_withdraw(self, amount: float) -> bool: ...
```

Python does **not** want Java-style accessors. If all a getter does is return an attribute, expose the
attribute; if it later needs logic, `@property` upgrades it without touching a single caller.

**Bad** :angry:

```python
class Account:
    def __init__(self, balance: float) -> None:
        self._balance = balance

    def get_balance(self) -> float:
        return self._balance

    def set_balance(self, balance: float) -> None:
        self._balance = balance
```

**Good** :smiley:

```python
class Account:
    def __init__(self, balance: float) -> None:
        self.balance = balance          # just an attribute


class InterestBearingAccount:
    def __init__(self, deposits: list[float]) -> None:
        self._deposits = deposits

    @property
    def balance(self) -> float:         # computed, but read like an attribute
        return sum(self._deposits)
```

When a constructor cannot express what it builds, hide it behind a classmethod whose name says what it makes:

```python
from datetime import date


class Money:
    def __init__(self, amount: float, currency: str) -> None:
        self.amount = amount
        self.currency = currency

    @classmethod
    def in_shillings(cls, amount: float) -> 'Money':
        return cls(amount, 'KES')


rent = Money.in_shillings(25_000)   # better than Money(25_000, 'KES')
```

**[⬆ back to top](#table-of-contents)**

### Pick-One-Word-per-Concept

---

Pick one word for one abstract concept and stay with it across the whole codebase. `fetch`, `retrieve` and
`get` as equivalent methods on different classes is a puzzle: the reader has to remember which class uses
which word, and every difference in wording suggests a difference in behaviour that is not there.

**Bad** :angry:

```python
class CustomerRepository:
    def fetch(self, customer_id: str) -> Customer: ...

class OrderRepository:
    def retrieve(self, order_id: str) -> Order: ...

class InvoiceRepository:
    def get(self, invoice_id: str) -> Invoice: ...
```

**Good** :smiley:

```python
class CustomerRepository:
    def get(self, customer_id: str) -> Customer: ...

class OrderRepository:
    def get(self, order_id: str) -> Order: ...

class InvoiceRepository:
    def get(self, invoice_id: str) -> Invoice: ...
```

Now the API is guessable: once you have used one repository you have used them all. A consistent lexicon is
a gift to every reader who comes after you.

The same holds for the objects themselves. Do not have a `Controller`, a `Manager` and a `Driver` in one
system if they all play the same role. Write down the vocabulary somewhere (a `CONTRIBUTING.md`, a glossary,
a docstring in `__init__.py`) and treat it as part of the design.

**[⬆ back to top](#table-of-contents)**

### Don't-Pun

---

The previous rule has a twin. Using one word for **two** concepts is a pun, and it is just as confusing as
using two words for one concept.

Suppose `add` in your codebase has always meant "return a new value by joining two existing values". Now you
need a method that puts a single item into a collection. Calling it `add` is a pun: the word is the same,
the semantics are not.

**Bad** :angry:

```python
class ShoppingCart:
    def add(self, other: 'ShoppingCart') -> 'ShoppingCart':
        """Combine two carts into a new one."""

    def add(self, item: Item) -> None:          # same word, different concept
        """Put one item into this cart."""
```

**Good** :smiley:

```python
class ShoppingCart:
    def merge(self, other: 'ShoppingCart') -> 'ShoppingCart':
        """Combine two carts into a new one."""

    def append(self, item: Item) -> None:
        """Put one item into this cart."""
```

Python makes the cost concrete: two methods with the same name in one class means the second one silently
replaces the first. Where you genuinely have one concept with several input shapes, use
`functools.singledispatchmethod` or an explicit classmethod, not a pun.

> Author names like a technical writer, not like a poet. The goal is a codebase where a reader can guess
> right without looking anything up.

**[⬆ back to top](#table-of-contents)**

### Use-Solution-Domain-Names

---

The people reading your code are programmers. Use the vocabulary of computer science, algorithms, patterns
and mathematics where it fits: it is precise, and it saves you from inventing a worse word for something that
already has a name.

**Bad** :angry:

```python
class JobHolderThatKeepsThingsInOrder:
    def put_in(self, job): ...
    def take_out(self): ...
```

**Good** :smiley:

```python
from queue import PriorityQueue


class JobQueue:
    def __init__(self) -> None:
        self._jobs: PriorityQueue = PriorityQueue()

    def enqueue(self, job: 'Job') -> None: ...
    def dequeue(self) -> 'Job': ...
```

A reader who knows what a priority queue is now understands the class in one line, including its performance
characteristics.

The same applies to design patterns. `AccountVisitor`, `RetryDecorator` and `ConnectionPool` all import a
whole chapter of shared understanding for free.

```python
class LoggingPaymentDecorator(Payment):     # says exactly what it is
    ...
```

Just make sure the name is true. Calling something a `Factory` when it is not one is disinformation of the
worst kind, because it is *confident* disinformation.

**[⬆ back to top](#table-of-contents)**

### Use-Problem-Domain-Names

---

When there is no computer science term for what you are doing, use the language of the business. Code that
speaks the domain's vocabulary lets a domain expert read it and spot a bug you cannot see.

**Bad** :angry:

```python
def process(record: dict) -> float:
    value = record['amount'] * 0.16
    return value
```

**Good** :smiley:

```python
from decimal import Decimal
from typing import Final

VAT_RATE: Final[Decimal] = Decimal('0.16')


def value_added_tax(invoice: 'Invoice') -> Decimal:
    return invoice.taxable_amount * VAT_RATE
```

An accountant can read the second version and tell you whether 16% is still the rate, and whether the base
should really be the taxable amount. They can do nothing with the first.

Getting the vocabulary right is a design activity, not a naming afterthought: the words the business uses
usually map onto the classes the system needs. If the business says "a policy is underwritten and then
issued", expect a `Policy` with `underwrite()` and `issue()`, not a `PolicyManager` with `handle_state()`.

> **The rule of thumb:** if the concept has a name in computer science, use it. If it does not, use the name
> the domain experts use. Never invent a third word.

**[⬆ back to top](#table-of-contents)**

### Add-Meaningful-Context

---

Very few names are meaningful on their own. `state` could be a US state, an HTTP status or a state machine's
current node. Most names need context, and the question is where to put it.

The weakest option is prefixing (see [Gratuitous Context](#gratuitous-context)). Better options, roughly in
order of strength:

**1. A well named enclosing function**

**Bad** :angry:

```python
def print_guess_statistics(candidate: str, count: int) -> None:
    if count == 0:
        number = 'no'
        verb = 'are'
        plural_modifier = 's'
    elif count == 1:
        number = '1'
        verb = 'is'
        plural_modifier = ''
    else:
        number = str(count)
        verb = 'are'
        plural_modifier = 's'
    print(f'There {verb} {number} {candidate}{plural_modifier}')
```

The three variables are only meaningful together, and you have to read the whole function to see that.

**2. A class that names the group**

**Good** :smiley:

```python
class GuessStatisticsMessage:
    def __init__(self, candidate: str, count: int) -> None:
        self._candidate = candidate
        self._count = count

    def __str__(self) -> str:
        return f'There {self._verb} {self._number} {self._candidate}{self._plural_modifier}'

    @property
    def _verb(self) -> str:
        return 'is' if self._count == 1 else 'are'

    @property
    def _number(self) -> str:
        return 'no' if self._count == 0 else str(self._count)

    @property
    def _plural_modifier(self) -> str:
        return '' if self._count == 1 else 's'
```

The class name supplies the context, so each part can have a short name and each part can be tested.

**3. Group related fields into a type instead of repeating a prefix**

**Bad** :angry:

```python
def ship(street: str, city: str, state: str, postal_code: str, country: str) -> None: ...
```

**Good** :smiley:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Address:
    street: str
    city: str
    state: str          # unambiguous now: it is part of an Address
    postal_code: str
    country: str


def ship(destination: Address) -> None: ...
```

The `Address` type gives `state` its meaning, shrinks the signature from five arguments to one (see
[Function Arguments](functions.md#function-arguments)) and gives you somewhere to put validation.

**4. The module and package**

`billing/invoice.py` needs no `Billing` prefix on `Invoice`. Let the import path carry the context:
`from billing.invoice import Invoice`.

> Only add context when the name genuinely needs it. `Address` does not need to become `CustomerAddress`
> unless you also have a `WarehouseAddress` to tell it apart from.

**[⬆ back to top](#table-of-contents)**

