---
name: python-development-database
description: Patterns for working with relational databases using metaorm. Use when defining tables, repositories, filters, database settings, or transaction logic.
---

## Rules
- Install the ORM layer with `pip install metaorm`. Requires Python `>=3.12`.
- Every database sub-system lives in its own package (conventionally named `database/` or `db/`). The package follows the repository pattern: one repository class per database table.
- The minimum package structure is:
  ```
  database/
      __init__.py
      repositories/
          __init__.py
          user.py
          ...
      filters.py       # only if filters are not imported from an external schema package
      settings.py      # only if RepositorySettings from metaorm needs extension
      cli.py           # optional, for management commands (create-tables, migrations, etc.)
  ```
- In `database/__init__.py` export the public API: repository classes, filter classes, settings, and any shared exceptions. Keep `__all__` as a tuple grouped by source module.

### Tables
- Table models inherit from `metaorm.BaseTable` and use `sqlmodel.Field` for column declarations. They are declared with `table=True`.
- Keep table classes in the same module as their repository or in a dedicated `tables.py` module when shared across multiple repositories. Prefer co-location with the repository.
- Use type unions for nullable columns: `id: int | None = Field(default=None, primary_key=True)`.
- Define relationships with `metaorm.Relationship` (re-exported from SQLModel). Always declare the back-populates attribute.
- Provide `from_item` and `to_item` class/instance methods when the repository uses a DTO.

Example `database/repositories/user.py`:
```python
from metaorm import BaseTable, Field, Relationship


class UserTable(BaseTable, table=True):
    __tablename__ = "users"

    id: int | None = Field(default=None, primary_key=True)
    name: str
    email: str = Field(unique=True)
    orders: list["OrderTable"] = Relationship(back_populates="user")
```

### Filters
- Filters inherit from `metaorm.BaseFilter` (re-exported from `pydantic-filters`). Filter fields are optional by default; simply declare the type without `| None = None`.
- Keep filters in `database/filters.py` when they are reused across the project, or inline in the repository module when they are private to that repository.
- Filter field names must match the table column names they filter.

Example `database/filters.py`:
```python
from metaorm import BaseFilter


class UserFilter(BaseFilter):
    name: str
    email: str


class OrderFilter(BaseFilter):
    user_id: int
    total: float
```

### Repositories
- Every repository inherits from `metaorm.BaseRepository` and is declared with keyword arguments `table=...` and `filter_=...`. Optionally add `dto=...` to enable automatic DTO mapping.
- Repository classes must specify `table=` and `filter_=` on the first class in the inheritance chain; subsequent subclasses can omit them if the parent already defined them.
- Keep each repository in its own module under `database/repositories/`. The module name matches the table name in singular snake_case.
- Provide a classmethod or factory function for repository instantiation when the project uses a shared `RepositoriesContainer`.
- Repository methods are async. The built-in CRUD methods are: `create_item`, `get_item`, `get_items`, `get_items_count`, `update_items`, `delete_items`.
- `get_items`, `update_items` return an `AsyncGenerator`. Collect results with list comprehension: `[item async for item in repo.get_items(...)]`.

Example `database/repositories/user.py`:
```python
from metaorm import BaseRepository

from project.database.filters import UserFilter
from .tables import UserTable


class UserRepository(BaseRepository, table=UserTable, filter_=UserFilter):
    pass
```

Example with DTO `database/repositories/user.py`:
```python
from pydantic import BaseModel

from metaorm import BaseRepository

from project.database.filters import UserFilter
from .tables import UserTable


class User(BaseModel):
    id: int | None = None
    name: str
    email: str


class UserRepository(BaseRepository, table=UserTable, filter_=UserFilter, dto=User):
    pass
```

### Settings
- If the default `metaorm.RepositorySettings` (fields: `dsn`, `pool_size`, `pool_recycle`, `pool_timeout`) is sufficient, import and use it directly. Do not create a local wrapper.
- If the project needs additional database-level configuration (e.g. migration settings, schema names, connection retries), define a subclass in `database/settings.py` that inherits from `metaorm.RepositorySettings`.
- Pass the settings instance to the repository constructor or to a shared `RepositoriesContainer`.

Example `database/settings.py`:
```python
from metaorm import RepositorySettings
from pydantic import Field


class DatabaseSettings(RepositorySettings):
    migration_path: str = "migrations"
    auto_migrate: bool = False
```

### Transactions
- Every CRUD method automatically runs inside a transaction. Do not wrap single CRUD calls in an explicit transaction unless you need to combine multiple operations atomically.
- Use `repository.transaction()` as an async context manager when you need to group multiple operations on the same repository into one atomic unit.
- Use `RepositoriesContainer` when a single atomic transaction must span multiple repositories. Retrieve repositories via `container.get_repository(RepositoryClass)` so they share the container's session context.
- `RepositoriesContainer.create_tables(*repository_classes)` creates tables for all given repositories sequentially using the shared engine.
- Use `container.transaction()` (or `repository.nested_transaction()`) for savepoints inside an existing transaction. An exception inside a nested transaction rolls back only the savepoint, leaving the outer transaction intact.

Example multi-repository transaction `database/transactions.py`:
```python
from metaorm import RepositoriesContainer, RepositorySettings

from .repositories.order import OrderRepository, OrderTable
from .repositories.user import UserRepository, UserTable


async def create_user_with_order(settings: RepositorySettings) -> None:
    container = RepositoriesContainer(settings=settings)
    user_repo = container.get_repository(UserRepository)
    order_repo = container.get_repository(OrderRepository)

    async with container.transaction():
        user = await user_repo.create_item(UserTable(name="Alice"))
        await order_repo.create_item(OrderTable(user_id=user.id, total=100.00))
```

### Pagination, sorting, and eager loading
- Use `metaorm.OffsetPagination` or `metaorm.PagePagination` for result pagination.
- Use `metaorm.BaseSort` for ordering. Pass it to `get_items` as `sort=BaseSort(sort_by="column", sort_by_order="asc|desc")`.
- Pass SQLAlchemy eager-loading options (`joinedload`, `selectinload`) to `get_items` via the `options` parameter.

Example `database/repositories/book.py`:
```python
from sqlalchemy.orm import joinedload

from metaorm import BaseFilter, BaseRepository, BaseSort, BaseTable, Field, OffsetPagination


class BookTable(BaseTable, table=True):
    __tablename__ = "books"

    id: int | None = Field(default=None, primary_key=True)
    title: str
    author_id: int = Field(foreign_key="authors.id")


class BookFilter(BaseFilter):
    title: str
    author_id: int


class BookRepository(BaseRepository, table=BookTable, filter_=BookFilter):
    pass


async def list_books(repository: BookRepository) -> list[BookTable]:
    pagination = OffsetPagination(offset=0, limit=20)
    sort = BaseSort(sort_by="title", sort_by_order="asc")
    return [
        item async for item in repository.get_items(
            pagination=pagination,
            sort=sort,
            options=[joinedload(BookTable.author)],
        )
    ]
```

### Exceptions
- Metaorm raises `DatabaseException` subclasses. Catch them at the service layer, not inside repositories.
- Available exception types: `NotFoundError`, `AlreadyExistsError`, `HaveNoSessionError`. All inherit from `DatabaseException`.
- Let SQLAlchemy `IntegrityError` propagate as `AlreadyExistsError` when it signals a unique constraint violation; the library already does this internally for `create_item`.

### CLI (optional)
- If the database package provides management commands (create tables, health check, seed data), place them in `database/cli.py`.
- CLI commands are async and accept the database settings as a parameter. They use the repository classes or `RepositoriesContainer` directly.
