---
name: python-development-stack
description: Technology stack decisions for Python services. Use when adding dependencies, choosing infrastructure, bootstrapping projects, or reviewing tool choices.
---

## Rules
- Use `uv` as the package manager for all Python dependency operations.
- Use `FastAPI` for REST API and HTTP services.
- Use `metaorm` for database access. It is built on top of `SQLModel` and `SQLAlchemy` and provides built-in CRUD, DTO mapping, pagination, sorting, and transaction management. Async drivers: `asyncpg` for PostgreSQL, `aiosqlite` for SQLite.
- Use `alembic` for database migrations.
- Use `facet` for hierarchical async service composition with dependency management and graceful shutdown.
- Use `typer` for CLI interfaces.
- Use `pydantic` for DTOs and data validation; use `pydantic-settings` for application configuration.
- Use `loguru` for logging.
- Use `pytest` with `pytest-asyncio` for testing.
- Use `pytest-cov` for coverage reporting.
- Use `faker` for generating test data.
- Use `pyyaml` for YAML parsing; use `pyyaml-include` for `!include` tags and `pyyaml-env-tag` for `!env` interpolation inside YAML config files.

