# Behind the Underscores EP10: Context Management (`__enter__`, `__exit__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep10-context-management-__enter__-__exit__/).

<br>


Have you ever opened a file in Python, wrote something, and forgot to close it? Maybe it didn’t break your program, but it’s not good practice. Leaving files or network connections open can cause resource leaks, meaning you’re using up system memory or leaving a file locked unnecessarily. That’s where context managers come in. They handle the “setup and teardown” automatically so you can focus on your logic without worrying about the cleanup.

This blog will guide you through:

- What a context manager is
- How `__enter__` and `__exit__` work
- Real-life use cases and examples
- How to write your own context managers both class-based and function-based

Let’s dive in!

## What Is a Context Manager?

A context manager is a Python object that properly manages resources like files, network connections, or database sessions. It makes sure things are set up when you enter a block of code and cleaned up when you leave it, even if something goes wrong.

You’ve already used one before:

```python
with open("myfile.txt", "w") as f:
    f.write("Hello, world!")
```

What this does behind the scenes:

1. Python calls `f = open(...)`, then `f.__enter__()`
2. It runs your `f.write(...)` inside the `with` block
3. When the block is done or crashes, it calls `f.__exit__()` to close the file

You didn’t have to write a `try`/`finally` block. Python cleaned up for you.

## The `__enter__` and `__exit__` Methods

To create a context manager yourself, you need a class that defines two special methods:

```python
class MyContext:
    def __enter__(self):
        # Setup code here
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # Cleanup code here
        pass
```

Let’s see this in action with a simple logger.

## Example 1: A Simple Logging Context Manager

```python
import time

class Timer:
    def __enter__(self):
        self.start = time.time()
        print("Starting the timer...")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        end = time.time()
        print(f"Elapsed time: {end - self.start:.2f} seconds")
```

Usage:

```python
with Timer():
    # Simulate work
    time.sleep(1.5)
```

**Output:**

```
Starting the timer...
Elapsed time: 1.50 seconds
```

Even if there’s an error inside the block, `__exit__` still runs which is great for cleanup.

## Real-Life Use Cases

Let’s take this a bit further. Here are some practical real-world problems you can solve with custom context managers.

### 1. **Automatically Closing Resources**

Imagine you're working with file handles, network sockets, or database connections. You need to ensure they're closed no matter what happens.

Instead of writing:

```python
db = connect_to_db()
try:
    do_something(db)
finally:
    db.close()
```

Use a context manager:

```python
with connect_to_db() as db:
    do_something(db)
```

### **2. Temporarily Change Working Directory**

You might want to run a script in a different folder temporarily and go back automatically.

```python
import os

class ChangeDirectory:
    def __init__(self, path):
        self.new_path = path
        self.original_path = os.getcwd()

    def __enter__(self):
        os.chdir(self.new_path)

    def __exit__(self, exc_type, exc_value, traceback):
        os.chdir(self.original_path)
```

Usage:

```python
print("Before:", os.getcwd())

with ChangeDirectory("/tmp"):
    print("Inside:", os.getcwd())

print("After:", os.getcwd())
```

It cleanly returns you to your original path. Great for file-heavy automation scripts.

### 3. **Thread Locking in Multithreading**

Working with `threading.Lock()`?

```python
import threading

lock = threading.Lock()

# Instead of this:
lock.acquire()
try:
    do_something()
finally:
    lock.release()

# Do this:
with lock:
    do_something()
```

The lock is automatically released after the block.

### 4. **Suppressing Output Temporarily**

Sometimes you use a noisy library that prints too much. You can silence it:

```python
import sys
import os
from contextlib import contextmanager

@contextmanager
def suppress_output():
    original_stdout = sys.stdout
    sys.stdout = open(os.devnull, 'w')
    try:
        yield
    finally:
        sys.stdout.close()
        sys.stdout = original_stdout
```

Usage:

```python
with suppress_output():
    print("This won't show up.")
```

This is handy when running external tools or verbose APIs.

### 5. **Retrying on Failure**

Want to retry a risky operation automatically?

```python
class Retry:
    def __init__(self, retries):
        self.retries = retries

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        if exc_type and self.retries > 0:
            self.retries -= 1
            return True  # Suppress the error and retry
        return False  # Let the exception propagate if out of retries
```

Wrap in a loop:

```python
while True:
    with Retry(3) as r:
        try:
            risky_operation()
            break
        except:
            if r.retries == 0:
                raise
```

You just built a mini fault-tolerant system!

## Final Thoughts

Context managers are one of Python’s most powerful but underused features. Once you start using them, you'll find dozens of places where they clean up your code and prevent bugs especially around resources, cleanup, and state changes.

**Use them when:**

- You need something to be cleaned up after use
- You're dealing with files, sockets, locks, or temporary state
- You want readable and bug-resistant code

Start small. Try writing one or two yourself. You’ll see how easy and useful they really are.