# Understanding Async Context Managers in Python

Check this post on my blog [here](https://hevalhazalkurt.com/blog/understanding-async-context-managers-in-python/).

<br>

When working with asynchronous code in Python, you're probably familiar with `async def`, `await`, and maybe even tools like `aiohttp` or `asyncpg`. But if you've ever wondered how to manage resources cleanly and safely in async code, then it's time to meet a powerful tool: the async context manager.

In this post, we’ll take a deep dive into what async context managers are, why they matter, and how to use them effectively in real-world backend applications.

## What Is a Context Manager Again?

Before we dive into the async version, let’s briefly recall what a context manager is in general. In Python, context managers are commonly used with the `with` statement to manage setup and teardown logic:

```python
with open('log.txt', 'w') as f:
    f.write("Logging something important.")
```

The file is automatically closed once the block ends even if an error occurs. This pattern is clean and safe.

## Why Do We Need Async Context Managers?

The traditional `with` block works fine for synchronous code. But what if your resource like a database connection, network socket, or HTTP session, needs to be handled asynchronously? That’s where async context managers come in. They’re used with `async with`, and they allow you to:

- Await asynchronous setup/teardown.
- Manage resources like connections in `async` code.
- Avoid race conditions and resource leaks.

## How Async Context Managers Work

You use `async with` to enter an asynchronous context:

```python
class AsyncLogger:
    async def __aenter__(self):
        print("Async Enter")
        return self

    async def __aexit__(self, exc_type, exc_val, tb):
        print("Async Exit")

async def main():
    async with AsyncLogger():
        print("Inside block")

asyncio.run(main())
```

**So what’s happening here?**

- `__aenter__`: Think of it as an async version of `__enter__`. You can `await` things here.
- `__aexit__`: Similar to `__exit__`, but `await`able.

## Backend Example 1: Managing an Async HTTP Session with aiohttp

Let’s say you’re fetching data from a REST API in an async backend service:

```python
import aiohttp
import asyncio

async def fetch_data():
    async with aiohttp.ClientSession() as session:
        async with session.get("https://api.example.com/data") as response:
            data = await response.json()
            print(data)

# asyncio.run(fetch_data())
```

**Why this is great:** 

- The session and request are both async context managers.
- They’re automatically closed and cleaned up.
- No need to manually close connections.

## Backend Example 2: Async Database Connection Pool

Let’s say you're building a FastAPI app that talks to PostgreSQL using `asyncpg`:

```python
import asyncpg
from fastapi import FastAPI

app = FastAPI()

@app.on_event("startup")
async def startup():
    app.state.pool = await asyncpg.create_pool(database="testdb")

@app.on_event("shutdown")
async def shutdown():
    await app.state.pool.close()

@app.get("/users")
async def get_users():
    async with app.state.pool.acquire() as conn:
        rows = await conn.fetch("SELECT * FROM users")
        return [dict(row) for row in rows]
```

**Why use `async with` for DB?**

- Automatically releases connections back to the pool.
- Prevents DB connection leaks.
- Keeps code clean and easy to maintain.

## Building Your Own Async Context Manager with `contextlib`

You can also build async context managers easily using `contextlib.asynccontextmanager`:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def temporary_file():
    print("Setting up async resource")
    yield "/tmp/temp.txt"
    print("Cleaning up async resource")

async def main():
    async with temporary_file() as path:
        print(f"Using temp file at {path}")

# asyncio.run(main())
```

This approach is simpler and cleaner than defining a full class with `__aenter__`/`__aexit__` when all you need is a quick setup/teardown pattern.

## Advanced Patterns

### Nesting Async Context Managers

You can nest multiple async contexts just like sync ones:

```python
async with aiohttp.ClientSession() as session:
    async with session.get(url) as response:
        data = await response.text()
```

Or combine them with `async with` and commas:

```python
async with manager1(), manager2():
    ...
```

### Combining with `async for`

```python
async with some_streaming_context() as stream:
    async for chunk in stream:
        await process(chunk)
```

Useful when reading big files or streaming API responses.

### Using Inside Dependency Injection (FastAPI Example)

```python
@asynccontextmanager
async def get_db():
    async with db_pool.acquire() as conn:
        yield conn

@app.get("/products")
async def read_products(db = Depends(get_db)):
    rows = await db.fetch("SELECT * FROM products")
    return rows
```

## Common Pitfalls

| **Pitfall** | **Explanation** |
| --- | --- |
| Forgetting `await` | `__aenter__` and `__aexit__` are coroutines. You must `await` them using `async with`. |
| Blocking calls inside `async with` | Avoid using synchronous (blocking) code inside async blocks. It kills concurrency. |
| Not releasing resources | If you don’t use `async with`, you might forget to close sessions, which can lead to memory or connection leaks. |

## Cheat Sheet

```python
# Basic Structure
class MyAsyncManager:
    async def __aenter__(self): ...
    async def __aexit__(self, exc_type, exc, tb): ...

# Shortcut with contextlib
@asynccontextmanager
async def my_manager():
    yield resource

# Usage
async with my_manager() as res:
    await do_something(res)
```

## Final Thoughts

Async context managers are a must-know tool for writing clean, safe, and efficient asynchronous code in modern Python. Whether you're dealing with HTTP clients, database pools, or temporary async resources, they help you stay in control without clutter.

If you're writing backend services with FastAPI, aiohttp, or Quart, mastering async context managers will make your architecture more robust and professional.