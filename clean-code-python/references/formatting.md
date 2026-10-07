## **Formatting**

## Table of Contents

- [Formatting](#formatting)
- [Automate it, then never discuss it again](#automate-it-then-never-discuss-it-again)
- [Vertical formatting](#vertical-formatting)
- [Horizontal formatting](#horizontal-formatting)
- [Team rules beat personal preference](#team-rules-beat-personal-preference)


---

Formatting is about communication, and communication is the professional developer's first order of
business. Code formatting matters far too much to ignore — and far too much to argue about, which is why the
right answer in Python is to **stop doing it by hand**.

### Automate it, then never discuss it again

```bash
pip install black ruff
black .          # formats every file the same way, no options to argue over
ruff check .     # catches unused imports, shadowed builtins, dead code
```

Add both to a pre-commit hook and to CI, and the entire class of "style feedback" disappears from code
review, leaving reviewers free to talk about design.

[PEP 8](https://peps.python.org/pep-0008/) is the baseline every Python tool implements: four spaces per
indent, `snake_case` for functions and variables, `PascalCase` for classes, `UPPER_SNAKE_CASE` for
constants, and a line limit (88 in `black`, 79 in strict PEP 8).

### Vertical formatting

**Files should be small.** A module of 200 lines is comfortable; one of 2,000 is a filing cabinet, not a
document.

**Blank lines separate concepts.** Each blank line is a visual cue that a new thought begins.

```python
import csv
from decimal import Decimal

TAX_RATE = Decimal('0.16')


def total_with_tax(subtotal: Decimal) -> Decimal:
    return subtotal + subtotal * TAX_RATE


def read_rows(path: str) -> list[list[str]]:
    with open(path) as handle:
        return list(csv.reader(handle))
```

**Related lines stay together.** Do not separate a variable from its first use with unrelated code.

**Vertical distance matters.** Declare a variable as close to its use as possible; keep a function near the
functions it calls (see [the Stepdown Rule](functions.md#the-stepdown-rule)); keep conceptually related functions close
even if neither calls the other.

**Newspaper metaphor.** A module should read like a newspaper article: the name at the top tells you the
topic, the first functions give the high-level story, and the details appear further down.

### Horizontal formatting

**Lines should be short enough to read without scrolling.** If a line is too long, the usual cause is too
much happening on it:

**Bad** :angry:

```python
return [transform(item) for item in fetch(source) if item.is_valid and item.created_at > cutoff and item.owner in allowed]
```

**Good** :smiley:

```python
def is_relevant(item: Item) -> bool:
    return item.is_valid and item.created_at > cutoff and item.owner in allowed


return [transform(item) for item in fetch(source) if is_relevant(item)]
```

**Use whitespace to show association.** Space around low-precedence operators, none around high:

```python
total = base_price + quantity*unit_price
```

**Do not align assignments in columns.** It looks tidy and it produces a diff on every line whenever one
name changes.

**Indentation shows hierarchy — do not collapse it.** Even when Python allows a one-liner:

**Bad** :angry:

```python
if not accounts: return None
```

**Good** :smiley:

```python
if not accounts:
    return None
```

### Team rules beat personal preference

A codebase should look like it was written by one person, not by a committee of individually tasteful
programmers. Agree the configuration once, put it in `pyproject.toml`, and let the tool enforce it:

```toml
[tool.black]
line-length = 88

[tool.ruff]
line-length = 88
select = ["E", "F", "I", "B"]     # errors, pyflakes, import order, bugbear
```

**[⬆ back to top](#table-of-contents)**

