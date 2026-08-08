---
name: python-development-layout
description: Folder structure and file organization for Python services. Use when creating a new service, reorganizing modules, or reviewing project structure.
---

## Rules
- Every repository must contain a `README.md`.
- Every repository must contain a `.gitignore` that excludes `.venv/`, `*.pyc`, and `__pycache__/`.
- Every repository must contain a `.dockerignore` with content similar to `.gitignore`.
- Every repository must contain a `Dockerfile` for containerized deployment.
- Every repository must contain a `pyproject.toml` created automatically by `uv init`.
- The Python package source folder lives in the repository root (e.g. `my_package/`), not under a `src/` directory.
- `pyproject.toml` must explicitly name the root package folder so the build system knows where the code lives.
- All tool configuration (tests, linters, formatters, type checkers, coverage, etc.) must live inside `pyproject.toml` under the `[tool.*]` tables. Do not use separate dot-files such as `.pylintrc`, `.flake8`, `setup.cfg`, or `tox.ini` for tool settings.
- The repository root may also contain a `tests/` folder and a `docs/` folder alongside the package folder.
- Inside the root package every sub-package is either a service package, an adapter package, or a library package. The structure of each must follow the conventions below.
- The root package must contain a `__main__.py` module. It is the executable entry point when the package is run with `python -m package_name`. Keep it minimal: import only from sibling modules (relative imports), call the factory that builds the CLI or the root service, and invoke it under `if __name__ == "__main__":`. No business logic lives here. Example:
  ```python
  from .cli import get_cli

  if __name__ == "__main__":
      cli = get_cli()
      cli()
  ```

### Service package layout
A package that exposes a runnable service must contain at minimum:
- `service.py` — the service class. It inherits from the async service mixin and implements `start`/`stop` lifecycle methods. The constructor accepts exactly one settings instance — the class declared in the same package's `settings.py`. Example:
  ```python
  import facet

  from .settings import MyServiceSettings

  class MyService(facet.AsyncioServiceMixin):
      def __init__(self, settings: MyServiceSettings):
          self._settings = settings

      async def start(self):
          # service startup logic
          pass
  ```
- `settings.py` — a Pydantic Settings class that declares the configuration schema for the service.
- `interfaces.py` or `base.py` — abstract ports (interfaces) that the service depends on. Consumers import from here, never from adapters.
- `fabric.py` — a factory module that instantiates the service and its collaborators. It receives settings and returns fully constructed objects.
- `__init__.py` — public API of the package. Exports only the types and functions intended for external use.
