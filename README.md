# Skills

Reusable agent skills. Install the entire directory of a skill, including its references.

| Skill | Purpose |
| --- | --- |
| [clean-code-python](clean-code-python/SKILL.md) | Apply the Python adaptation of Clean Code whenever writing, editing, auditing, refactoring, or reviewing Python code or a Python project. |
| [french-docstring](french-docstring/SKILL.md) | Understand code, audit existing declaration documentation, and add or update IDE-readable documentation in French. |

## Installation

Copy a skill directory into `~/.agents/skills/` for personal use across projects, or into
`.agents/skills/` in a target repository for project use.

## Source maintenance

The local `Clean-Code-Python-skill/` directory is an independent development checkout and is excluded
from this collection's Git history. The installable `clean-code-python/` directory is a copy of its
`skills/clean-code-python/` package. Refresh that copy after changing the source skill.
