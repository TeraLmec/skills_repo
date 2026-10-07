## Introduction

## Table of Contents

- [Introduction](#introduction)
- [Translation](#translation)


---

Software is read far more often than it is written. A line of code is typed once, but it will be read by
you, by your teammates and by the person who maintains the system long after you have moved on. Every
minute you invest in making that line obvious is paid back many times over.

**Clean code** is code that is easy to read, easy to reason about and cheap to change. Robert C. Martin
describes it as code that "always looks like it was written by someone who cares". It is not about
cleverness, it is about communication.

This repository adapts the ideas from
[_Clean Code_](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882) to Python.
A few things to keep in mind while reading:

- **This is not a style guide.** Formatting is a solved problem in Python; hand it to
  [`black`](https://black.readthedocs.io/), [`ruff`](https://docs.astral.sh/ruff/) or
  [PEP 8](https://peps.python.org/pep-0008/) and spend your energy on design instead.
- **These are guidelines, not laws.** Every rule here has a cost. A rule applied without judgement
  produces code that is technically compliant and practically unreadable.
- **Python is not Java.** Many "Clean Code" examples in the wild are ceremonial because they come from a
  language without first class functions, default arguments, tuples or duck typing. Where Python offers a
  simpler tool than a design pattern, the notes say so.
- **Refactoring is continuous.** Nobody writes clean code on the first pass. You write code that works,
  then you clean it while the tests keep you honest.

The chapters that follow move from the smallest unit of design to the largest: names, then functions, then
data, then classes, then the SOLID principles that govern how classes depend on one another, and finally
the cross cutting concerns of testing, concurrency, error handling and comments.

**[⬆ back to top](#table-of-contents)**

## **Translation**

---

This guide is a Python adaptation of Robert C. Martin's *Clean Code*.
