# AI Skills

This repository contains skills and a superpower used by AI agents when working on this codebase.

All documents are written in English. They describe technology choices, project structure, architecture, and coding conventions.

## Index

| File | Type | Description |
|------|------|-------------|
| [`skills/python-development/SKILL.md`](skills/python-development/SKILL.md) | Superpower | High-level philosophy, when to apply, and skill routing map |
| [`skills/python-development-stack/SKILL.md`](skills/python-development-stack/SKILL.md) | Skill | Technology stack: Python, FastAPI, SQLModel, facet, typer, loguru, uv |
| [`skills/python-development-layout/SKILL.md`](skills/python-development-layout/SKILL.md) | Skill | File and folder organization, service and adapter package layout |
| [`skills/python-development-architecture/SKILL.md`](skills/python-development-architecture/SKILL.md) | Skill | Ports and Adapters, Monolith-over-Microservices, facet hierarchy |
| [`skills/python-development-code-style/SKILL.md`](skills/python-development-code-style/SKILL.md) | Skill | Imports, `__init__.py`, wildcard bans, block grouping |
| [`skills/python-development-settings/SKILL.md`](skills/python-development-settings/SKILL.md) | Skill | Pydantic Settings, BaseModel, YAML/JSON loading, validators |
| [`skills/python-development-cli/SKILL.md`](skills/python-development-cli/SKILL.md) | Skill | Typer composition, callback boot, deferred imports, nesting |

## Usage

When an agent starts working on a task, it should load the superpower first.
The superpower contains a decision tree that points to the relevant granular skills.

## Customizing

Edit the files in `skills/` to match your actual preferences, frameworks, and team conventions.
