# The Power of yield from in Python Generators

Check this post on my blog [here](https://hevalhazalkurt.com/blog/the-power-of-yield-from-in-python-generators/).

<br>

Python generators are one of the best tools in your toolbox when you want to write efficient, readable, and memory-friendly code. But there's a hidden gem inside Python's generator system that makes things even smoother: the **`yield from`** statement.

In this post, we'll walk through:

1. What `yield from` is and how it works.
2. How to use it to build clean, modular generator utilities.
3. A practical example: streaming logs in real-time from a Python backend using FastAPI + `yield from`.

Let’s get started!

## A Quick Refresher on Generators

A **generator** is a function that allows you to return a sequence of values one at a time, using the `yield` keyword.

Example:

```python
def count_up_to(n):
    i = 1
    while i <= n:
        yield i
        i += 1
```

This doesn’t return a list, it gives back a **generator object** that you can loop through:

```python
for number in count_up_to(3):
    print(number)
```

Output:

```
1
2
3
```

This kind of lazy evaluation is perfect when working with large files, data streams, or pipelines, you only load what you need, when you need it.

## Enter `yield from`

Now imagine you want to delegate part of your generator logic to another generator. You could do this:

```python
def wrapper():
    for value in count_up_to(3):
        yield value
```

But Python offers a cleaner, smarter way:

```python
def wrapper():
    yield from count_up_to(3)
```

Boom. Just one line and it works exactly the same. `yield from` basically forwards all values from another iterable or generator into your current one.

## Example 1: Flattening a Nested List

Let’s flatten this list:

```python
data = [1, [2, 3], [4, [5, 6]], 7]
```

We want: `1, 2, 3, 4, 5, 6, 7`

Here’s how:

```python
def flatten(items):
    for item in items:
        if isinstance(item, list):
            yield from flatten(item)
        else:
            yield item
```

Usage:

```python
for val in flatten([1, [2, 3], [4, [5, 6]], 7]):
    print(val)
```

`yield from` makes recursive generators a breeze!

## Example 2: Combining Multiple Sources

Imagine you’re pulling data from different sources:

```python
def fruits():
    yield from ["apple", "banana"]

def veggies():
    yield from ["carrot", "daikon"]

def everything():
    yield from fruits()
    yield from veggies()
```

`everything()` will yield all items from both functions, without any manual loops. This makes your code super modular.

---

## Real-World Use Case: Streaming Logs with `yield from` and FastAPI

Let’s now take what we’ve learned and apply it to a real-world backend use case: serving live logs via an HTTP endpoint.

### Why Use Generators for Log Streaming?

In backends, you often need to stream logs to the frontend. This can be for:

- Real-time debugging
- Monitoring
- Admin dashboards

But log files can be huge, and loading them all at once is a bad idea.

That’s where generators (and `yield from`) come in:

- Stream logs line-by-line
- Use almost no memory
- React to updates as they happen

### Step-by-Step: Build a Live Log Streaming API

### 1. Install FastAPI and Uvicorn

```bash
pip install fastapi uvicorn
```

### 2. Create the Streaming App

```python
# main.py
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import time
import os

app = FastAPI()

def tail_log_file(filepath):
    """A generator that yields new lines from a log file, like `tail -f`"""
    with open(filepath, "r") as f:
        # Move to end of file
        f.seek(0, os.SEEK_END)
        while True:
            line = f.readline()
            if not line:
                time.sleep(0.5)  # Wait for new lines
                continue
            yield line

def log_streamer():
    """Wrapper that could combine multiple sources using `yield from`"""
    # In the future, you could use yield from other sources here.
    yield from tail_log_file("app.log")

```

### 3. Add the FastAPI Endpoint

```python
@app.get("/logs")
def stream_logs():
    return StreamingResponse(log_streamer(), media_type="text/plain")
```

### 4. Run the App

```bash
uvicorn main:app --reload
```

Then visit http://localhost:8000/logs and watch the logs stream in real time!

## Why `yield from` Matters Here

You might just stream from one source today, but tomorrow you might want to stream logs from:

- A file
- A subprocess (e.g. `subprocess.Popen`)
- A queue (e.g. Kafka, Redis)
- Another API

With `yield from`, you can combine and delegate streams without breaking your structure.

Example:

```python
def log_streamer():
    yield from tail_log_file("app.log")
    yield from tail_log_file("error.log")
```

This modular design makes it easy to grow your app in the future.

## Final Thoughts

Python’s `yield from` is more than a syntax shortcut—it’s a powerful tool for writing clean, modular, and memory-efficient generators.

We saw how it helps simplify:

- Recursive utilities (like list flattening)
- Combining multiple generators
- Real-time streaming in production-ready backends

And with frameworks like FastAPI, you can use it to build elegant, high-performance data streaming endpoints with just a few lines of code.