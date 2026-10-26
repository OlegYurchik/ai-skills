---
name: python-development-fastapi
description: Patterns for building a FastAPI HTTP service inside the facet service tree. Use when creating REST endpoints, routers, or the api/rest service package.
---

## Rules
- The FastAPI service lives in a sub-package named `api/rest/` inside the root package. It is a regular facet service: it has `service.py`, `settings.py`, and `__init__.py`.
- Provide a custom `UvicornServer` that disables signal handlers so the parent facet orchestrator controls startup and shutdown. Example:
  ```python
  import uvicorn

  class UvicornServer(uvicorn.Server):
      def install_signal_handlers(self):
          pass
  ```
- The service class inherits from `facet.AsyncioServiceMixin`. Its constructor accepts the settings instance declared in the same package's `settings.py`. Keep the constructor minimal; heavy initialization happens in `start()`.
- The `start()` method builds the FastAPI application asynchronously, configures uvicorn, and adds the server coroutine to the facet task list with `self.add_task(server.serve())`.
- Extract app construction into an `async` `_build_app()` method so router registration and dependency wiring can use `await`.
- Register routers in `_setup_app()` or `_build_app()` using `app.include_router(...)`. Group related endpoints under routers with a `prefix`.
- The settings class inherits from `pydantic.BaseModel` (not `BaseSettings`). It declares the minimum required for the HTTP layer: `host`, `port`, and references to other subsystem settings when the REST layer wires them directly.

### Minimal complete example

`api/rest/settings.py`:
```python
from pydantic import BaseModel, conint


class RESTAPISettings(BaseModel):
    host: str = "127.0.0.1"
    port: conint(ge=1, le=65535) = 8000
```

`api/rest/service.py`:
```python
import facet
import fastapi
import uvicorn

from . import handlers, users
from .settings import RESTAPISettings


class UvicornServer(uvicorn.Server):
    def install_signal_handlers(self):
        pass


class RESTAPIService(facet.AsyncioServiceMixin):
    def __init__(self, settings: RESTAPISettings):
        self._settings = settings

    async def start(self):
        app = await self._build_app()
        config = uvicorn.Config(
            app=app,
            host=self._settings.host,
            port=self._settings.port,
        )
        server = UvicornServer(config)
        self.add_task(server.serve())

    async def _build_app(self) -> fastapi.FastAPI:
        app = fastapi.FastAPI()
        await self._setup_app(app)
        return app

    async def _setup_app(self, app: fastapi.FastAPI):
        app.get("/health")(handlers.health)
        app.include_router(users.get_users, prefix="/users")
```

### Router and handler structure

- Endpoints must be RESTful. The URL path structure maps directly to the package structure inside `api/rest/`.
- Each URL segment becomes a sub-package. For example `/users/{id}` lives in `api/rest/users/`.
- Every handler sub-package contains at minimum:
  - `router.py` — defines `get_router() -> fastapi.APIRouter` and wires each route to a handler function.
  - `handlers.py` — plain async functions that implement the endpoint logic.
  - `schemas.py` — Pydantic request and response models shared by the handlers in this package.
  - `__init__.py` — exports only `get_router`.
- Optional but common additional files:
  - `dependencies.py` — FastAPI dependency functions reused across handlers in the package.
  - `exceptions.py` — custom HTTP exceptions for this domain.

### Dependencies

- Declare reusable dependencies in `dependencies.py` inside the handler sub-package. Dependencies are plain functions or async functions that return a value needed by endpoints (database session, current user, settings, etc.).
- Use `fastapi.Depends` in `router.py` to wire dependencies to routes. Avoid calling dependencies directly in handlers.
- Keep dependencies stateless; if they need configuration, accept it as an argument and use a closure or a callable class when wiring.

Example `api/rest/users/dependencies.py`:
```python
import fastapi

from project.api.rest.exceptions import HTTPUnauthorized
from project.api.rest.dependencies import database


async def current_user(
    database: DatabaseService = fastapi.Depends(database),
    token: str = fastapi.Depends(tokens.oauth2_scheme),
) -> User:
    db_token = await database.get_token(token)
    if db_token is None:
        raise HTTPUnauthorized()
    return await database.get_user(filter_=UserFilter(id=db_token.user_id))
```

Example `api/rest/users/schemas.py`:
```python
from pydantic import BaseModel


class UserResponse(BaseModel):
    user_id: int
    requested_by: str
```

Example `api/rest/users/handlers.py` consuming a dependency:
```python
import fastapi

from . import dependencies, schemas


async def get_user(
    id: int,
    user: User = fastapi.Depends(dependencies.current_user),
) -> schemas.UserResponse:
    return schemas.UserResponse(user_id=id, requested_by=user.username)
```

### List endpoints

- Use a shared `ListResponse` schema for endpoints that return collections. It is defined once in `api/rest/schemas.py` and reused across packages.
- Extract pagination and search via a reusable `pagination` dependency declared in `api/rest/dependencies.py`.

Example `api/rest/schemas.py`:
```python
from typing import Any
from pydantic import BaseModel


class ListResponse(BaseModel):
    total: int
    next: str | None
    previous: str | None
    data: list[Any]


class UserResponse(BaseModel):
    ...


class UsersResponse(ListResponse):
    data: list[UserResponse]
```

Example `api/rest/dependencies.py`:
```python
import fastapi
from pydantic import BaseModel


class Pagination(BaseModel):
    cursor: str | None = None
    limit: int = 20
    search: str | None = None
    order_by: str | None = None


def pagination(
        cursor: str | None = fastapi.Query(default=None),
        limit: int = fastapi.Query(default=20, ge=1),
        search: str | None = fastapi.Query(default=None),
        order_by: str | None = fastapi.Query(default=None)
) -> PaginationParams:
    return PaginationParams(cursor=cursor, limit=limit, search=search)
```

Example `api/rest/users/handlers.py` using the list schema and pagination dependency:
```python
import fastapi

from api.rest.dependencies import PaginationParams, pagination
from . import schemas


async def get_users(
    pagination: PaginationParams = fastapi.Depends(pagination),
) -> schemas.UsersResponse:
    ...
```

Example `api/rest/users/router.py`:
```python
import fastapi

from . import handlers


def get_router() -> fastapi.APIRouter:
    router = fastapi.APIRouter()
    router.add_api_route(path="/", methods=["GET"], endpoint=handlers.list_users)
    router.add_api_route(path="/", methods=["POST"], endpoint=handlers.create_user)
    router.add_api_route(path="/{id}", methods=["GET"], endpoint=handlers.get_user)
    router.add_api_route(path="/{id}", methods=["PUT"], endpoint=handlers.update_user)
    router.add_api_route(path="/{id}", methods=["DELETE"], endpoint=handlers.delete_user)
    return router
```

Example `api/rest/users/__init__.py`:
```python
from .router import get_router

__all__ = (
    # router
    "get_router",
)
```

- Do not construct `fastapi.APIRouter` instances at import time outside `get_router()`. The router is created only when `get_router()` is called during `_setup_app()`.
```
