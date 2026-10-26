---
name: python-development-tests
description: Guidelines for writing and organizing tests in Python services. Use when creating or modifying tests, fixtures, or test configuration.
---

## Rules

- All tests are located in the `tests` folder at the repository root.
- The test directory structure mirrors the structure of the tested package. For example, if a module is at `project/database/service.py`, the corresponding test must be at `tests/database/test_service.py`.
- Do **not** test protected methods or functions (`_method`, `_function`).
- Do **not** rely on or verify internal implementation details in tests.
- Use only public classes, objects, functions, and methods — and test only them.
- Do **not** use `monkeypatch` or any form of patching in tests.
- If internal behavior of an object needs to be changed, pass a mock object via the constructor or function argument (dependency injection).
- Actively use fixtures and `parametrize` for tests.
- All fixtures are stored in `conftest.py`.
- Every test package has its own `conftest.py`.
- Make tests asynchronous where applicable.
