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
