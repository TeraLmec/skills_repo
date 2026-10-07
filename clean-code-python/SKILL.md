---
name: clean-code-python
description: >
  Apply this skill whenever writing, editing, auditing, refactoring, or reviewing
  Python code or a Python project. Use it for every Python task, including new
  features, bug fixes, tests, scripts, APIs, project structure, configuration, and
  maintenance, as well as Python work in mixed-language repositories. Apply the
  repository's Python adaptation of Robert C. Martin's Clean Code.
---

# Clean Code Python

Use this guide for every task that writes, edits, audits, refactors, or reviews Python code or a Python
project. Apply it before making changes or forming review findings, even when the user has not explicitly
asked for Clean Code guidance. In a mixed-language project, apply it to the Python work and its relevant
project context.

## Apply the guide

1. Establish the requested outcome and scope. For a review or audit, produce findings; make changes when
   implementation is requested. Inspect the relevant code, callers, project instructions, Python version,
   and existing tests and tooling.
2. Read [Introduction](references/introduction.md) once when first using this skill in a task. Use the
   reference table below to select the chapters relevant to the work. For a narrow task, use each
   chapter's contents to find the applicable sections. Read their explanations and examples together.
3. Apply the relevant principles to the requested work. Preserve intended behaviour and public contracts
   during refactoring; implement requested behaviour changes explicitly. Follow project conventions and
   the user's requirements. As the introduction explains, these principles are guidelines to apply with
   judgement in Python.
4. Verify the result with checks appropriate to the change and the project's existing tooling. For
   reviews and audits, ground findings in the actual code and explain their concrete consequences.
5. Report the changes or findings, the reasoning that matters, and the verification performed. State
   remaining limitations when a check could not be completed.

Using the skill on every Python task does not require reading every chapter on every task. Choose the
references that bear on the work, and revisit that choice if the scope expands. For a project-wide audit,
cover the applicable chapters across the project and read concurrency guidance when concurrent work is
present.

## References

The references preserve the original tutorial's wording, stories, Good/Bad labels, and examples. Examples
remain in the same sections as their instructions. Some examples intentionally demonstrate bad code, and
some are illustrative fragments: use the surrounding explanation to interpret them and verify any code
adapted into the target project.

| When working on | Read |
| --- | --- |
| The guide's purpose and how to apply its principles in Python | [Introduction](references/introduction.md) |
| Names for variables, functions, classes, modules, or domain concepts | [Naming Things](references/naming.md) |
| Function boundaries, arguments, conditionals, side effects, queries, or duplication | [Functions](references/functions.md) |
| Data representations, encapsulation, object relationships, or transfer objects | [Objects and Data Structures](references/objects-and-data-structures.md) |
| Class size, cohesion, organisation for change, or abstraction boundaries | [Classes](references/classes.md) |
| Choosing which SOLID principle applies | [SOLID Principles](references/solid-principles.md) |
| Separating responsibilities or changing a processing workflow | [Single Responsibility Principle](references/single-responsibility.md) |
| Extending behaviour, strategies, or decorators | [Open/Closed Principle](references/open-closed.md) |
| Inheritance, subtype behaviour, or contracts | [Liskov Substitution Principle](references/liskov-substitution.md) |
| Interfaces that impose unrelated methods on clients or implementations | [Interface Segregation Principle](references/interface-segregation.md) |
| Dependencies, injection, replaceable infrastructure, or test seams | [Dependency Inversion Principle](references/dependency-inversion.md) |
| Tests, assertions, fakes, or behaviour-preserving verification | [Testing](references/testing.md) |
| Async code, threads, processes, or shared state | [Concurrency](references/concurrency.md) |
| Exceptions, missing values, validation, or resource cleanup | [Error Handling](references/error-handling.md) |
| Layout, whitespace, formatter settings, or lint configuration | [Formatting](references/formatting.md) |
| Comments, docstrings, intent, or explanations embedded in code | [Comments](references/comments.md) |

For Python project structure or configuration changes, select references according to their effect:
organisation and dependencies, tooling and formatting, tests, or runtime behaviour. Keep the work within
the requested scope.

## Review and audit output

Prioritise actionable findings by their impact on correctness, maintainability, and the requested goal.
For each finding, identify the location, explain the specific problem and its consequence, and recommend
a concrete improvement. Connect it to the relevant principle when that helps the reader understand it.
Distinguish confirmed problems from suggestions and uncertainties. If there are no actionable findings,
say so and describe any limits of the review.
