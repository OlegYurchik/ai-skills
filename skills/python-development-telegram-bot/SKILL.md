---
name: python-development-telegram-bot
description: Patterns for building a Telegram bot service using aiogram inside the facet service tree. Use when creating bot handlers, routers, callbacks, inline keyboards, pagination, or the telegram_bot service package.
---

## Rules
- The Telegram bot service lives in a sub-package named `telegram_bot/` inside the root package. It is a regular facet service: it has `service.py`, `settings.py`, and `__init__.py`.
- Use `aiogram` as the Telegram Bot API framework.
- The settings class inherits from `pydantic.BaseModel` (not `BaseSettings`). It declares the minimum required: `method` (polling or webhook) and `token`.
- Provide separate settings subclasses for each method:
  - `TelegramBotPollingSettings` — adds `timeout`.
  - `TelegramBotWebhookSettings` — adds `root_url`, `root_path`, `port`, `secret_access_key`.
- The base service class inherits from `facet.AsyncioServiceMixin`. Its constructor accepts the settings instance declared in the same package's `settings.py`. Keep the constructor minimal; heavy initialization happens in `start()`.
- The base service creates `aiogram.Bot` and `aiogram.Dispatcher` in `__init__` or `start()`.
- Extract router registration into a `get_router()` function (or `setup_dispatcher()` when the dispatcher itself needs external dependencies). Handlers live in `handlers.py`, router wiring lives in `router.py`.
- Register the router in the service using `dispatcher.include_router(get_router())`.
- Use `aiogram.fsm.state.StatesGroup` for multi-step conversations.
- Use `aiogram.filters.callback_data.CallbackData` for typed callback data on inline buttons. Every callback action gets its own `CallbackData` subclass with a unique `prefix`.

### Minimal complete example

`telegram_bot/settings.py`:
```python
import enum

from pydantic import BaseModel


class TelegramBotMethodEnum(str, enum.Enum):
    POLLING = "polling"
    WEBHOOK = "webhook"


class TelegramBotSettings(BaseModel):
    method: TelegramBotMethodEnum = TelegramBotMethodEnum.POLLING
    token: str = "123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11"
```

`telegram_bot/polling/settings.py`:
```python
from typing import Literal

from pydantic import PositiveInt

from project.telegram_bot.enums import TelegramBotMethodEnum
from project.telegram_bot.settings import TelegramBotSettings


class TelegramBotPollingSettings(TelegramBotSettings):
    method: Literal[TelegramBotMethodEnum.POLLING] = TelegramBotMethodEnum.POLLING
    timeout: PositiveInt = 10  # in seconds
```

`telegram_bot/webhook/settings.py`:
```python
from typing import Literal

from pydantic import AnyHttpUrl, conint, constr

from project.telegram_bot.enums import TelegramBotMethodEnum
from project.telegram_bot.settings import TelegramBotSettings


class TelegramBotWebhookSettings(TelegramBotSettings):
    method: Literal[TelegramBotMethodEnum.WEBHOOK] = TelegramBotMethodEnum.WEBHOOK

    root_url: AnyHttpUrl
    root_path: str = "/"
    port: conint(ge=1, le=65535) = 8000
    secret_access_key: constr(pattern=r"^\w{32}$") = "webhooksuperpupersecretaccesskey"
```

`telegram_bot/service.py`:
```python
import aiogram
import facet

from .settings import TelegramBotSettings


class BaseTelegramBotService(facet.AsyncioServiceMixin):
    def __init__(
        self,
        settings: TelegramBotSettings,
    ):
        self._settings = settings
        self._dispatcher = aiogram.Dispatcher()
        self._bot = aiogram.Bot(token=settings.token)

    async def start(self):
        self.add_task(self.listen_events())

    async def listen_events(self):
        raise NotImplementedError
```

`telegram_bot/polling/service.py`:
```python
from project.telegram_bot.base import BaseTelegramBotService


class TelegramBotPollingService(BaseTelegramBotService):
    async def listen_events(self):
        await self._bot.delete_webhook()
        await self._dispatcher.start_polling(
            self._bot,
            handle_signals=False,
        )
```

`telegram_bot/webhook/service.py`:
```python
from typing import Any

import aiogram
import fastapi
import uvicorn
import yarl

from project.telegram_bot.base import BaseTelegramBotService


class UvicornServer(uvicorn.Server):
    def install_signal_handlers(self):
        pass


class TelegramBotWebhookService(BaseTelegramBotService):
    async def listen_events(self):
        webhook_url = yarl.URL(str(self._settings.root_url)) / self._settings.root_path.lstrip("/")
        await self._bot.set_webhook(
            url=str(webhook_url),
            secret_token=self._settings.secret_access_key,
        )

        await self.get_server().serve()

    def get_server(self) -> uvicorn.Server:
        app = fastapi.FastAPI(
            root_url=self._settings.root_url,
            root_path=self._settings.root_path,
        )
        app.post("")(self.handler)

        config = uvicorn.Config(app=app, host="0.0.0.0", port=self._settings.port)
        return UvicornServer(config)

    async def handler(
        self,
        update: dict[str, Any],
        x_telegram_bot_api_secret_token: str | None = fastapi.Header(None),
    ):
        if x_telegram_bot_api_secret_token != self._settings.secret_access_key:
            raise fastapi.HTTPException(
                status_code=fastapi.status.HTTP_403_FORBIDDEN,
                detail="Forbidden.",
            )

        await self._dispatcher.feed_webhook_update(
            bot=self._bot,
            update=aiogram.types.Update(**update),
        )
```

### Router and handler structure

- Each domain or feature gets its own router package under `telegram_bot/ui/` (or `telegram_bot/handlers/`).
- Every router sub-package contains at minimum:
  - `router.py` — defines `get_router() -> aiogram.Router` and wires each route to a handler function.
  - `handlers.py` — plain async functions that implement the bot logic.
  - `callback_data.py` — `CallbackData` subclasses for inline buttons used by this feature.
  - `states.py` — `StatesGroup` classes for FSM flows used by this feature.
  - `__init__.py` — exports only `get_router`.

Example `telegram_bot/ui/router.py`:
```python
import aiogram
from aiogram.filters import Command

from . import handlers
from .callback_data import ItemCallbackData
from .states import FormState


def get_router() -> aiogram.Router:
    router = aiogram.Router()

    router.message.register(handlers.start_handler, Command("start"))
    router.message.register(handlers.ask_name_handler, FormState.WAITING_FOR_NAME)
    router.callback_query.register(
        handlers.item_callback_handler,
        ItemCallbackData.filter(),
    )

    return router
```

Example `telegram_bot/ui/__init__.py`:
```python
from .router import get_router

__all__ = ("get_router",)
```

- Do not construct `aiogram.Router` instances at import time outside `get_router()`. The router is created only when `get_router()` is called during service setup.

### Dispatcher setup

- If the router needs external dependencies (repositories, services, etc.), expose a `setup_dispatcher(dispatcher, **deps)` function that attaches middlewares and includes routers.

Example `telegram_bot/ui/dispatcher.py`:
```python
import aiogram

from . import get_router
from .middlewares import AuthMiddleware, TypingMiddleware


def setup_dispatcher(
    dispatcher: aiogram.Dispatcher,
    users_repository: UsersRepository,
):
    auth_middleware = AuthMiddleware(users_repository=users_repository)
    typing_middleware = TypingMiddleware()

    for middleware in (auth_middleware, typing_middleware):
        dispatcher.update.outer_middleware(middleware)

    router = get_router()
    dispatcher.include_router(router)
```

### Middlewares

- Keep middlewares stateless. If they need configuration, accept it as an argument and pass it when wiring in `setup_dispatcher()`.

Example `telegram_bot/ui/middlewares/typing.py`:
```python
import asyncio
import contextlib
from collections.abc import Awaitable, Callable

import aiogram


class TypingMiddleware(aiogram.BaseMiddleware):
    async def __call__(
        self,
        handler: Callable[[aiogram.types.TelegramObject, dict[str, Any]], Awaitable[Any]],
        event: aiogram.types.TelegramObject,
        data: dict[str, Any],
    ):
        if not isinstance(event, aiogram.types.Message):
            return await handler(event, data)

        bot = data.get("bot")
        chat_id = event.chat.id
        async with send_typing_action(bot=bot, chat_id=chat_id):
            return await handler(event, data)


@contextlib.asynccontextmanager
async def send_typing_action(bot: aiogram.Bot, chat_id: int, delay: float = 3):
    async def background_task():
        while True:
            await bot.send_chat_action(chat_id=chat_id, action="typing")
            await asyncio.sleep(delay)

    task = asyncio.create_task(background_task())
    try:
        yield
    finally:
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass
```

### Callbacks and Inline Keyboards

- Use `aiogram.filters.callback_data.CallbackData` subclasses for every inline button action. The `prefix` must be unique across the whole bot.
- For actions that share a paginated list, create a base `PaginatorCallbackData` with a `page` field and inherit from it.

Example `telegram_bot/ui/callback_data.py`:
```python
import enum
import uuid

from aiogram.filters.callback_data import CallbackData


class PaginatorCallbackData(CallbackData, prefix=""):
    page: int = 1


class MenuCallbackData(CallbackData, prefix="menu"):
    pass


class ItemsCallbackData(PaginatorCallbackData, prefix="items"):
    pass


class ItemActionEnum(str, enum.Enum):
    SHOW = "show"
    DELETE = "delete"


class ItemCallbackData(CallbackData, prefix="item"):
    item_id: uuid.UUID
    action: ItemActionEnum = ItemActionEnum.SHOW
```

- Build inline keyboards with `aiogram.utils.keyboard.InlineKeyboardBuilder`.
- Pack callback data with `.pack()`. Unpack inside handlers with `MyCallbackData.unpack(callback_query.data)`.

Example inline keyboard builder in a handler:
```python
from aiogram import types
from aiogram.utils.keyboard import InlineKeyboardBuilder

from .callback_data import ItemCallbackData, ItemsCallbackData


async def show_items_handler(message: types.Message):
    keyboard = InlineKeyboardBuilder()
    for item in items:
        keyboard.row(types.InlineKeyboardButton(
            text=item.name,
            callback_data=ItemCallbackData(item_id=item.id).pack(),
        ))
    keyboard.row(types.InlineKeyboardButton(
        text="Next page",
        callback_data=ItemsCallbackData(page=2).pack(),
    ))
    await message.reply(text="Items:", reply_markup=keyboard.as_markup())
```

### Lists and Pagination

- Use a shared `PaginatorCallbackData` base for every paginated list.
- The maximum number of list items per page is **5**.
- In the handler or logic layer, unpack the callback to read `page`, compute `total`, `pages_count`, and render a single navigation row.
- The navigation row contains up to 3 buttons in this order:
  1. **"◀️"** — shown only when `page > 1`.
  2. **"{current_page}/{pages_count}"** — shown only when `pages_count > 1`.
  3. **"▶️"** — shown only when `page < pages_count`.
- Navigation buttons use the same `CallbackData` subclass with an incremented or decremented `page`.

Example pagination logic inside a handler or `logic.py`:
```python
from aiogram import types
from aiogram.utils.keyboard import InlineKeyboardBuilder

from .callback_data import ItemsCallbackData, ItemCallbackData

PER_PAGE = 5


async def render_item_list(
    message: types.Message | None,
    callback_query: types.CallbackQuery | None,
    page: int,
    total: int,
    items: list[Item],
):
    pages_count = total // PER_PAGE + int(bool(total % PER_PAGE))

    keyboard = InlineKeyboardBuilder()
    for item in items[:PER_PAGE]:
        keyboard.row(types.InlineKeyboardButton(
            text=item.name,
            callback_data=ItemCallbackData(item_id=item.id).pack(),
        ))

    navigation_row = []
    if page > 1:
        navigation_row.append(types.InlineKeyboardButton(
            text="◀️",
            callback_data=ItemsCallbackData(page=page - 1).pack(),
        ))
    if pages_count > 1:
        navigation_row.append(types.InlineKeyboardButton(
            text=f"{page}/{pages_count}",
            callback_data="null",
        ))
    if page < pages_count:
        navigation_row.append(types.InlineKeyboardButton(
            text="▶️",
            callback_data=ItemsCallbackData(page=page + 1).pack(),
        ))
    if navigation_row:
        keyboard.row(*navigation_row)

    reply_markup = keyboard.as_markup()

    if message is not None:
        await message.reply(text="Items:", reply_markup=reply_markup)
    elif callback_query is not None:
        await callback_query.message.edit_text(text="Items:")
        await callback_query.message.edit_reply_markup(reply_markup=reply_markup)
```

- Keep the handler thin: extract keyboard building and pagination math into a `logic.py` module or a dedicated builder function.
