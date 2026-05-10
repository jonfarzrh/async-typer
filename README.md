# async-typer

[Typer](https://github.com/tiangolo/typer) with first-class async support:
`async def` commands and callbacks work alongside regular sync ones via the
same `@app.command()` decorator — no second API to remember — plus lifecycle
event handlers for setting up and tearing down async resources.

## Features

- **One decorator, sync or async** — `@app.command()` and `@app.callback()`
  accept both regular and `async def` functions. The wrapper is transparent;
  Typer's `--help`, option parsing, and type conversion all work as normal.
- **Shared event loop across the command lifecycle** — startup handlers,
  the command body, and shutdown handlers all run on the same
  [`asyncio.Runner`](https://docs.python.org/3/library/asyncio-runner.html),
  so async resources created on startup (connection pools, HTTP sessions,
  etc.) remain usable by the command and by shutdown.
- **Fully typed** — ships with a `py.typed` marker and strict type hints.
- **Drop-in replacement** — re-exports Typer's public API, so
  `from async_typer import Option, Argument, echo, ...` works without a
  second import line.

## Installation

```bash
pip install async-typer
# or
uv add async-typer
```

Requires Python 3.11+.

## Quick start

```python
from async_typer import AsyncTyper

app = AsyncTyper()


@app.command()
def sync_hello(name: str = "world") -> None:
    print(f"hi {name}")


@app.command()
async def async_hello(name: str = "world") -> None:
    # await anything you need here
    print(f"hello {name}")


if __name__ == "__main__":
    app()
```

## Async callbacks

```python
@app.callback()
async def main(verbose: bool = False) -> None:
    if verbose:
        print("verbose mode")
```

## Lifecycle event handlers

Register `startup` and `shutdown` hooks, sync or async. They run on the
same event loop as the command body, so shared async resources stay alive
across the whole invocation:

```python
import httpx

app = AsyncTyper()
state: dict[str, httpx.AsyncClient] = {}


async def open_client() -> None:
    state["client"] = httpx.AsyncClient()


async def close_client() -> None:
    await state["client"].aclose()


app.add_event_handler("startup", open_client)
app.add_event_handler("shutdown", close_client)


@app.command()
async def fetch(url: str) -> None:
    response = await state["client"].get(url)
    print(response.status_code)
```

The shutdown handler runs even if the command raises — use it to release
resources unconditionally.

## Migrating from 0.1.x

The separate `async_command` / `async_callback` decorators still work but
emit `DeprecationWarning`. Replace them with the unified `command` /
`callback`, which auto-detect `async def`:

```python
# before
@app.async_command()
async def foo(): ...

# after
@app.command()
async def foo(): ...
```

## Development

This repo uses [`uv`](https://github.com/astral-sh/uv),
[`ruff`](https://github.com/astral-sh/ruff), and
[`ty`](https://github.com/astral-sh/ty).

```bash
uv sync --dev
uv run pytest
uv run ruff check .
uv run ty check
```

## License

MIT — see [LICENSE.txt](LICENSE.txt).
