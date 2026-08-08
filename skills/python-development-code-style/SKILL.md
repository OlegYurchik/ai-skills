---
name: python-development-code-style
description: Code conventions, typing, testing habits, and engineering principles. Use when writing, reviewing, or refactoring Python code.
---

## Rules
- Follow PEP 8 for all Python code. Use `ruff` for automatic formatting and linting.
- Every package `__init__.py` must declare the public API explicitly with `__all__`. `__all__` is always a tuple, even when it contains a single element. Use inline comment group headers to separate imports by source module.
- Inside `__init__.py` import only from sibling modules. Do not place executable code.
- Group related names together in `__all__` and mirror the grouping with comment headers. Example:
  ```python
  from .base import BaseService, ServiceMixin
  from .dto import ServiceInput, ServiceOutput
  from .exceptions import ServiceError, ValidationError
  from .fabric import get_service
  from .settings import ServiceSettings

  __all__ = (
      # base
      "BaseService",
      "ServiceMixin",
      # dto
      "ServiceInput",
      "ServiceOutput",
      # exceptions
      "ServiceError",
      "ValidationError",
      # fabric
      "get_service",
      # settings
      "ServiceSettings",
  )
  ```
- Wildcard imports (`from module import *`) are forbidden in every module, including `__init__.py`. Always import explicit names.
- Within a package, import sibling modules using relative imports (`from .module import Name`). When importing from a different top-level package or from a parent package that is not a direct sibling, use an absolute import (`from project_name.module import Name`).
- Group imports in every module into three blocks separated by a blank line:
  1. Standard library (`import pathlib`, `from typing import Self`, etc.).
  2. Third-party packages (`import yaml`, `from pydantic import Field`, etc.).
  3. Project-local imports (`from .services import MyService`, `from microclaw.settings import Settings`, etc.).
- Within each block, place `from ... import ...` lines first, then `import ...` lines. Sort all lines alphabetically. Example:
  ```python
  import json
  import pathlib
  from typing import Self

  import yaml
  from pydantic import Field
  from pydantic_settings import BaseSettings

  from .services import MyService
  from microclaw.settings import Settings
  ```
- Functions and methods must not return tuples of multiple values. Returning a tuple is allowed only in exceptional cases (e.g. unpacking a well-known pair such as `(key, value)`). When a function needs to return more than one piece of data, define a Pydantic `BaseModel` for the result or reuse an existing model. This makes the return type self-documenting, enables IDE autocomplete, and keeps field names stable during refactoring.
- Do not use `from __future__ import annotations`. Use explicit forward-reference strings (e.g. `"MyClass"`) when necessary.
- Do not use `if TYPE_CHECKING:` blocks. Import the types you need at the top level. Deferred imports inside `TYPE_CHECKING` hide dependencies and break runtime introspection.
- Do not abbreviate variable or parameter names. Write them out in full so the intent is obvious without context. Examples of forbidden abbreviations: `idx` (use `index`), `dep` (use `dependency`), `cfg` (use `config` or `configuration`), `msg` (use `message`), `err` (use `error`), `resp` (use `response`). The same rule applies to class attributes, function parameters, and local variables.
