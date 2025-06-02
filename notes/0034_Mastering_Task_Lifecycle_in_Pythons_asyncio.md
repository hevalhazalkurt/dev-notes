# Mastering Task Lifecycle in Python’s asyncio

Check this post on my blog [here](https://hevalhazalkurt.com/blog/mastering-task-lifecycle-in-pythons-asyncio/).

<br>

## Why Understanding the Task Lifecycle Matters

If you've ever written an `async def` function and called `await` on it, congrats you’ve already touched the surface of Python’s `asyncio` event loop. But what makes async Python really powerful is how you can create, manage, monitor, and cancel concurrent tasks.

If you're building backend systems like:

- a background job runner,
- a microservice handling multiple API calls at once,
- or a web scraper firing off 500 requests per second

…understanding the full task lifecycle is critical to keeping things fast, clean, and reliable.

## Part 1 → What is an asyncio Task?

**Basic Concept**

An `asyncio.Task` is an object that wraps a coroutine and schedules it to run on the event loop. This is how you make a coroutine actually run “concurrently” with other coroutines.

```python
import asyncio

async def say_hello():
    await asyncio.sleep(1)
    print("Hello!")

async def main():
    task = asyncio.create_task(say_hello())  # This schedules the coroutine
    print("Task started...")
    await task  # Waits for the task to finish

asyncio.run(main())
```

**What’s going on here?**

- `create_task()` puts the coroutine into the event loop and returns a `Task` object immediately.
- You can continue doing other things before `await task` finishes.
- Once the task completes, it prints `"Hello!"`.

### Real-Life Use Case → Parallel API Requests

Suppose you’re calling multiple APIs for a backend dashboard:

```python
async def fetch_cpu():
    await asyncio.sleep(2)
    return "CPU usage: 55%"

async def fetch_memory():
    await asyncio.sleep(1)
    return "Memory usage: 70%"

async def main():
    cpu_task = asyncio.create_task(fetch_cpu())
    mem_task = asyncio.create_task(fetch_memory())

    results = await asyncio.gather(cpu_task, mem_task)
    print(results)

asyncio.run(main())

# ['CPU usage: 55%', 'Memory usage: 70%']
```

**What’s good here?**

- Tasks are run in parallel, not one after the other.
- `gather()` waits for both to complete and collects results.
- This pattern is common in microservices fetching multiple internal or external resources.

## Part 2 → Task States and the Lifecycle

**The Life of a Task**

Here’s what a typical task lifecycle looks like:

1. Created: `create_task()` is called.
2. Scheduled: Task is submitted to the event loop.
3. Running: Event loop picks it up and starts running it.
4. Waiting: Task hits `await` and suspends until the result is ready.
5. Done: Task returns a result or raises an exception.
6. Cancelled: Task is manually or automatically cancelled.

You can monitor this process:

```python
task = asyncio.create_task(my_coroutine())
print(task.done())  # False

await task
print(task.done())  # True
```

## Part 3 → Task Cancellation, Killing a Task Properly

**Graceful Cancellation**

Sometimes you need to cancel a task. Maybe it’s taking too long, the user stopped the operation, or you're shutting down the app.

```python
async def slow_op():
    try:
        await asyncio.sleep(10)
        return "done"
    except asyncio.CancelledError:
        print("Cancelled!")
        raise

async def main():
    task = asyncio.create_task(slow_op())
    await asyncio.sleep(1)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("Caught cancellation in main")

asyncio.run(main())
```

**What just happened?**

- `task.cancel()` signals cancellation.
- Inside `slow_op()`, the `sleep()` is interrupted and raises `CancelledError`.
- You must handle it, or the task dies abruptly.
- Re-raising the exception is good practice, but you can also swallow it if needed.

**Common Mistake**

If your coroutine doesn’t hit an `await`, cancellation won’t work.

```python
async def tight_loop():
    while True:
        pass  # BAD: no await = can't be interrupted!
```

This will freeze the event loop. Always `await` inside long-running loops, or use `asyncio.sleep(0)` to yield control.

## Part 4 → Preventing Cancellation with `asyncio.shield()`

In some cases, you may want to protect a task from being cancelled even if the outer logic is cancelled like from a user hitting Ctrl+C, a timeout, etc.

**When Would You Use This?**

Imagine you’re saving important data to disk or committing to a database. You don’t want it interrupted halfway through, even if the main task is cancelled.

Let’s look at that shielded task below.

```python
import asyncio

async def important_save():
    try:
        await asyncio.sleep(3)
        print("Save completed!")
    except asyncio.CancelledError:
        print("Save was cancelled!")
        raise

async def main():
    task = asyncio.create_task(important_save())
    try:
        await asyncio.wait_for(asyncio.shield(task), timeout=1)
    except asyncio.TimeoutError:
        print("Timeout, but save is still running...")

    await task  # Now we wait again to ensure it finishes

asyncio.run(main())
```

**What’s happening?**

- `wait_for(..., timeout=1)` cancels the waiting, but not the task inside `shield()`.
- Even after timeout, the task keeps running and eventually completes.
- This is great for backend systems doing non-interruptible work like:
    - database commits
    - billing operations
    - transactional file writes

## Part 5 → Setting Timeouts with `asyncio.wait_for()`

You often want to enforce timeouts on certain tasks to prevent them from hanging forever. For example, a service call that should respond within 5 seconds.

```python
async def long_task():
    await asyncio.sleep(10)
    return "done"

async def main():
    try:
        result = await asyncio.wait_for(long_task(), timeout=3)
    except asyncio.TimeoutError:
        print("Task took too long and was cancelled.")

asyncio.run(main())
```

**Behind the scenes…**

- `wait_for()` starts a timer, and if it hits the timeout, it cancels the coroutine.
- If you need to retry or fallback, you can wrap it in retry logic.

### Real-Life Use → API Timeout Handling

```python
async def fetch_from_microservice():
    await asyncio.sleep(6)
    return "response"

async def main():
    try:
        resp = await asyncio.wait_for(fetch_from_microservice(), timeout=5)
    except asyncio.TimeoutError:
        resp = "Fallback: using cache or default"
    print(resp)

asyncio.run(main())
```

## Part 6 → Managing Multiple Tasks with `TaskGroup` (Python 3.11+)

Starting in Python 3.11, `asyncio.TaskGroup` helps you group and manage tasks to make sure they:

- all complete, or
- all cancel if one fails.

**TaskGroup Basics**

```python
import asyncio

async def worker(name, delay):
    await asyncio.sleep(delay)
    print(f"{name} done")

async def main():
    async with asyncio.TaskGroup() as tg:
        tg.create_task(worker("task1", 1))
        tg.create_task(worker("task2", 2))
    print("All tasks completed.")

asyncio.run(main())
```

**Why It Matters**

- Easier error handling: if one task crashes, the others are cancelled.
- Cleaner syntax than managing `create_task()` + `gather()` manually.
- Ideal for microservices firing parallel backend requests or workers.

## Part 7 → Graceful Shutdowns, Killing Tasks Cleanly

When shutting down a backend server or background system, you need to:

1. Cancel all pending tasks
2. Wait for them to clean up
3. Exit the event loop safely

```python
import asyncio

async def long_task():
    try:
        while True:
            print("Running...")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print("Task got cancelled, cleaning up...")
        await asyncio.sleep(1)
        print("Cleanup done.")

async def main():
    task = asyncio.create_task(long_task())

    await asyncio.sleep(3)
    task.cancel()
    await task

asyncio.run(main())
```

**What’s happening?**

- The task runs an infinite loop as many servers or daemons do.
- When it gets cancelled, it exits the loop, does cleanup, then exits.
- This is how you avoid leaving files half-written or DB transactions open.

### Real-World Backend Scenario → FastAPI Cleanup Hook

When using FastAPI or a similar framework, you can plug cleanup logic into the lifespan event:

```python
@app.on_event("shutdown")
async def shutdown_event():
    for task in pending_tasks:
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass
```

## Summary: Task Lifecycle in a Nutshell

| **Stage** | **Description** |
| --- | --- |
| Created | `create_task()` is called, coroutine is wrapped |
| Scheduled | Task enters the event loop |
| Running | Coroutine starts executing |
| Awaiting | Task pauses on `await` |
| Done | Task finishes or raises |
| Cancelled | Task is interrupted gracefully |

### Tools and Patterns You Should Know

- `create_task()` → Schedule async work
- `await` → Yield control to the event loop
- `wait_for()` → Add timeout protection
- `shield()` → Prevent unwanted cancellation
- `TaskGroup` → Group tasks, handle failure gracefully
- Cancellation → Handle `CancelledError` properly

In backend systems, understanding the `asyncio` task lifecycle is not just a nice-to-have, it’s a must-have for building resilient, concurrent systems. From protecting your important writes to managing thousands of async calls, these techniques let you write Python that’s fast, clean, and predictable.