---
name: python-development-cli
description: CLI design patterns using typer. Use when building, nesting, or extending command-line interfaces.
---

## Rules
- Every package that exposes CLI commands contains a `cli.py` module with a `get_cli() -> typer.Typer` factory function. Do not create the Typer instance at import time.
- The root package CLI (`cli.py` in the root package) is the composition point. It calls `cli.add_typer(get_subpackage_cli(), name="subpackage")` to mount sub-package CLI trees as subcommands.
- The root CLI defines a single `callback` function that runs before any command. It is responsible for shared bootstrap: load settings, configure logging, and store the settings object in `ctx.obj["settings"]` so every sub-command can access it.

Example root CLI `cli.py`:
```python
import pathlib

import typer
from loguru import logger

from .service import ApplicationService
from .settings import RootSettings


def callback(
    ctx: typer.Context,
    env_path: pathlib.Path | None = typer.Option(
        None, "--env", "-e",
        help="Environment variables file location",
    ),
    config_path: pathlib.Path | None = typer.Option(
        None, "--config", "-c",
        help="Config file location",
    ),
):
    ctx.obj = {}
    settings = RootSettings.load(
        env_prefix="APP__",
        env_file=env_path,
        config_file=config_path,
    )
    ctx.obj["settings"] = settings
    logger.configure(handlers=[{"sink": sys.stderr, "level": settings.logging.level}])


def run(ctx: typer.Context):
    settings: RootSettings = ctx.obj["settings"]
    service = ApplicationService(settings=settings)
    asyncio.run(service.run())


def get_cli() -> typer.Typer:
    from .agents import get_cli as get_agents_cli
    from .cron import get_cli as get_cron_cli

    cli = typer.Typer()
    cli.callback()(callback)
    cli.command(name="run")(run)
    cli.add_typer(get_agents_cli(), name="agents")
    cli.add_typer(get_cron_cli(), name="cron")
    return cli
```

- Sub-package CLI commands receive `ctx: typer.Context` as the first parameter and read settings via `ctx.obj["settings"]`.

Example sub-package `cli.py`:
```python
import asyncio

import typer


def run(
    ctx: typer.Context,
    name: str | None = typer.Argument(default=None),
    debug: bool = typer.Option(False, "-d", "--debug"),
):
    # Deferred imports — heavy project modules are imported inside the command body
    from mypackage.resolver import DependencyResolver
    from mypackage.settings import RootSettings
    from mypackage.channels.tui import TUIChannel, TUIChannelSettings

    settings: RootSettings = ctx.obj["settings"]
    resolver = DependencyResolver(settings=settings)
    channel = TUIChannel(settings=TUIChannelSettings(debug=debug), ...)
    asyncio.run(channel.run())


def get_cli() -> typer.Typer:
    cli = typer.Typer()
    cli.command(name="run")(run)
    return cli
```

- Each CLI command is a plain Python function decorated with `cli.command(name="...")`. Keep command functions small: parse arguments, create a resolver or service from settings, and run async code with `asyncio.run()`. No business logic inside the command function.
- The root package `__main__.py` calls `get_cli()` and runs it. Example:
```python
from .cli import get_cli

if __name__ == "__main__":
    cli = get_cli()
    cli()
```
