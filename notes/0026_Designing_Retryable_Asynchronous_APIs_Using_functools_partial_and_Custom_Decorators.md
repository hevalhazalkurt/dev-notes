# Designing Retryable Asynchronous APIs Using `functools.partial` and Custom Decorators

Check this post on my blog [here](https://hevalhazalkurt.com/blog/designing-retryable-asynchronous-apis-using-functoolspartial-and-custom-decorators/).

<br>

In the world of building reliable backend systems, things break. APIs timeout, services become temporarily unavailable, and database queries may fail due to load. One common and essential solution to this is a retry mechanism, a pattern that automatically re-attempts a failed operation after a delay.

In this blog post, we’re going to explore how to build retryable asynchronous APIs using Python. We’ll walk through how to use `functools.partial` in combination with custom async decorators to create flexible, reusable retry logic. Let’s dive in.

## Why Retry Logic Matters

Let’s say you’re calling an external payment API or fetching analytics from a third-party provider. These services might fail intermittently. If you just let those exceptions bubble up, you’re either forcing your users to refresh the page or retry manually, which is a bad experience.

A retry decorator can give your system some resilience:

- Try again automatically if an exception occurs.
- Add delay, jitter, or exponential backoff.
- Log what failed and when it’s trying again.

## Starting Simple: Retry with Async Function

Let’s start with a basic version of an async retry decorator:

```python
import asyncio
import random
from functools import partial

async def retry(f, tries=3, delay=1):
    for attempt in range(tries):
        try:
            return await f()
        except Exception as e:
            if attempt == tries - 1:
                raise
            print(f"Attempt {attempt+1} failed: {e}. Retrying in {delay}s...")
            await asyncio.sleep(delay)
```

This works if you pass an async function with no arguments. But that’s a big limitation. What if you want to pass arguments to the function?

## Pre-binding Function Arguments with `functools.partial`

The `functools.partial` tool allows you to bind arguments to a function in advance.

```python
from functools import partial

async def greet(name):
    print(f"Hello, {name}!")

# Create a version of greet where name is pre-filled
partial_greet = partial(greet, "Alice")
await partial_greet()  # Hello, Alice!
```

This is exactly what we need to make our retry logic generic. We’ll use `partial` to wrap any function and its arguments into a single callable.

## Making a Reusable Retry Decorator

Now we’ll build a proper decorator that supports:

- Any async function with arguments
- Customizable retry parameters (delay, tries, backoff, jitter)

```python
from decorator import decorator  

@decorator
async def retry_decorator(f, *fargs, tries=3, delay=1, backoff=1, jitter=0, **fkwargs):
    _tries, _delay = tries, delay
    wrapped = partial(f, *fargs, **fkwargs)

    while _tries:
        try:
            return await wrapped()
        except Exception as e:
            _tries -= 1
            if not _tries:
                raise
            print(f"Retrying after error: {e}. Next attempt in {_delay}s")
            await asyncio.sleep(_delay + random.uniform(0, jitter))
            _delay *= backoff
```

Now, use it like this:

```python
@retry_decorator(tries=5, delay=2, backoff=2, jitter=1)
async def fetch_remote_data():
    # Simulate a flaky operation
    if random.random() < 0.7:
        raise Exception("Temporary failure")
    return {"status": "success"}
```

## Wrapping a DB Query or HTTP Call

Imagine you’re querying a third-party analytics service:

```python
@retry_decorator(tries=4, delay=1, backoff=2, jitter=0.5)
async def fetch_analytics(account_id):
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.example.com/data/{account_id}")
        response.raise_for_status()
        return response.json()
```

Even if the service fails a couple of times, your user never notices. It retries silently under the hood.

## How `partial` Helps

The magic of `partial()` is that we don’t have to worry about how many arguments the wrapped function has or what they are. We just pass them into `partial`, and it gives us a single callable to run in our retry loop.

This makes the decorator very generic and safe to use across different parts of the codebase.

## A Few Use Cases

- **Logging**: Pass a logger to log retries and errors.
- **Custom Exceptions**: Only retry certain errors (e.g., timeouts).
- **Max Delay**: Clamp exponential backoff to avoid long waits.

```python
@retry_decorator(tries=5, delay=1, backoff=2, jitter=(0.5, 1.5))
async def get_data():
    # maybe fails due to network
```

For jitter, you can pass a range tuple. The decorator could support `random.uniform(min, max)`.

## When to Use and When Not To

Use retry decorators for:

- External HTTP APIs
- Cache misses + refresh logic
- Transient DB/network errors

Don’t use them when:

- The failure is a programming bug
- You expect high throughput because retrying can increase load
- You want transactional guarantees like DB write consistency

## Wrap-up

Retry logic is one of those things that every backend system needs, but it's easy to overcomplicate. Using `functools.partial` lets you build flexible decorators that work with any async function, no matter its signature. By combining it with a custom async decorator, you can write production-grade retry logic that’s reusable, testable, and easy to read.