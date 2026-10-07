## **Comments**

## Table of Contents

- [Comments](#comments)
- [Explain yourself in code](#explain-yourself-in-code)
- [Bad comments](#bad-comments)
- [Good comments](#good-comments)


---

> **Don't comment bad code — rewrite it.** — Brian W. Kernighan and P. J. Plauger

A comment is a failure to express yourself in code. Sometimes it is a necessary failure, but it is never a
success to be proud of, because comments are not compiled, not tested and not maintained. The code moves on;
the comment stays and starts lying.

### Explain yourself in code

**Bad** :angry:

```python
# check if the employee is eligible for full benefits
if employee.flags & HOURLY_FLAG and employee.age > 65:
    ...
```

**Good** :smiley:

```python
if employee.is_eligible_for_full_benefits():
    ...
```

It usually takes only a few seconds of thought to replace a comment with a function or a variable whose name
says the same thing — and that name gets checked by tests and updated by refactoring tools.

### Bad comments

**Redundant comments** — the code already said it, twice as fast:

```python
i += 1      # increment i
```

**Misleading comments** — a comment that was true once. This is the worst kind, because readers trust it.

**Commented-out code** — delete it. Git remembers; nobody else does, and everyone who sees it is afraid to
remove it.

```python
# def old_calculation(x):
#     return x * 1.15      # from 2019, may still be needed?
```

**Journal comments** — a changelog at the top of every file. Version control does this better.

```python
# 2019-01-04  KM  Added tax handling
# 2020-06-11  VJ  Fixed rounding
```

**Noise comments** — comments that say nothing at all:

```python
class Account:
    """The Account class."""

    def __init__(self) -> None:
        """The constructor."""
```

**Position markers and closing-brace comments** — `# ---- helpers ----` is a sign a module should be split.

**Attributions** — `# added by Kasozi`. Git blame knows.

**Too much information** — do not paste the RFC into a docstring; link to it.

### Good comments

Some comments earn their place:

**Legal comments** — a licence header, kept short.

**Intent — the "why", not the "what"**:

```python
# Sort by descending price first: the pricing API returns ties in a random
# order, and the test-suite needs a stable ordering to compare against.
items.sort(key=lambda item: (-item.price, item.sku))
```

**Warning of consequences**:

```python
@pytest.mark.slow      # takes ~40s: it builds the full search index
def test_full_reindex(): ...
```

**Clarification of something you cannot change**:

```python
assert response.status_code == 202     # the vendor returns 202, not 201, on create
```

**`TODO` comments** — acceptable as a marker for work that cannot be done now, as long as they are scanned
and cleared regularly:

```python
# TODO(kasozi): remove once the v1 pricing endpoint is decommissioned (JIRA-142).
```

**Docstrings on public APIs** — for anything another team or another program will import, a docstring is
documentation, not a comment, and it belongs there:

```python
def value_added_tax(invoice: Invoice) -> Decimal:
    """Return the VAT owed on ``invoice``.

    The rate is applied to the taxable amount only; zero-rated lines are
    excluded. See the Finance Act, section 5(2).

    Raises:
        NegativeAmount: if the invoice total is below zero.
    """
```

Note what that docstring does **not** do: it does not restate the signature. Types are the parameter
documentation; the prose is for the rules a reader cannot infer.

> **Rule of thumb:** if a comment explains *what* the code does, delete it and fix the code. If it explains
> *why* the code is the way it is, keep it — that information exists nowhere else.

**[⬆ back to top](#table-of-contents)**

