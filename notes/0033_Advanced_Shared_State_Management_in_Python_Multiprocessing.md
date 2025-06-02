# Advanced Shared State Management in Python Multiprocessing

Check this post on my blog [here](https://hevalhazalkurt.com/blog/advanced-shared-state-management-in-python-multiprocessing/).

<br>

When you're building Python applications that do a lot of CPU-heavy work, `multiprocessing` is often the way to go. But if you've ever needed to share data between multiple processes, you know it can get tricky. Unlike threads, processes don’t share memory by default. Each one runs in its own space. That makes communication and shared state more complex. But don't worry. Python gives us a few powerful tools to manage shared state in multiprocessing environments.

This post will start from the basics and go deep into managing shared state effectively and safely, using real backend-style examples. Let's dive in.

## What Is Multiprocessing?

Before we get too deep into managing shared state, let’s clear up what multiprocessing actually means because it’s easy to confuse it with multithreading.

In simple terms, multiprocessing is about running multiple processes at the same time, each with its own Python interpreter and memory space. It’s Python’s way of getting around the Global Interpreter Lock (GIL), which normally stops multiple threads from running Python code in parallel on multiple CPU cores.

So why should you care?

Well, if you're doing heavy-duty computations like image processing, machine learning data prep, or crunching logs in a backend system, you can actually take full advantage of your CPU cores using multiprocessing. That means faster results and better performance.

Here’s the key takeaway:

- Threads share memory but can't run Python code in true parallel.
- Processes don’t share memory by default, but they run in real parallel.

Python’s `multiprocessing` module gives you tools to spin up processes, share data between them if needed, and coordinate their work. It's like having a team of workers instead of a single one trying to do everything.

In backend systems, you might use multiprocessing to:

- Run multiple request handlers that do CPU-heavy tasks
- Preprocess large data batches in parallel
- Offload background processing like resizing images or aggregating logs

Coming up, we’ll look at how to manage the shared data between these processes because as soon as your processes need to talk to each other or work on shared state, things get a bit more complicated.

But don’t worry, we’ll handle that step by step. 

## So, Why Shared State is Hard in Multiprocessing

In Python, the `multiprocessing` module creates separate processes. Each process has its own Python interpreter and memory space. That’s great for avoiding GIL issues in CPU-bound tasks but it also means you can't share normal variables across processes.

Let’s look at a quick example:

```python
from multiprocessing import Process

def increment(counter):
    counter += 1

if __name__ == "__main__":
    count = 0
    p = Process(target=increment, args=(count,))
    p.start()
    p.join()
    print(count)  # Output will still be 0!
```

Each process receives a copy of the data. So changing `counter` inside the new process doesn’t affect the original. So how do we fix this?

## Method 1: `multiprocessing.Value` and `Array`

**What They Are:**

- `Value` lets you share a single primitive like an integer or boolean.
- `Array` lets you share an array of primitives.

**When to Use:**

- For simple numeric counters, flags, or lists of fixed-length data.

**Example: Shared Counter Between Workers**

```python
from multiprocessing import Process, Value
import time

def worker(counter):
    for _ in range(100):
        time.sleep(0.01)
        with counter.get_lock():
            counter.value += 1

if __name__ == "__main__":
    counter = Value('i', 0)  # 'i' means integer
    processes = [Process(target=worker, args=(counter,)) for _ in range(4)]

    for p in processes:
        p.start()
    for p in processes:
        p.join()

    print("Final counter:", counter.value)  # Expect around 400
```

- `Value` holds a shared integer.
- `.get_lock()` is used to make sure only one process updates it at a time (critical section).

You might use this in a backend service that counts the number of successful data processing tasks completed across multiple worker processes.

## Method 2: `multiprocessing.Manager`

**What It Is:**

A `Manager` object spawns a server process that holds Python objects like `dict`, `list`, etc. Other processes interact with those via proxy.

**When to Use:**

- You need to share high-level objects like dicts or lists.
- Multiple processes need to read/write complex state.

**Example: Shared Dictionary for Caching Results**

```python
from multiprocessing import Process, Manager
import time

def cache_result(cache, key, value):
    time.sleep(1)  # Simulate long operation
    cache[key] = value

if __name__ == "__main__":
    with Manager() as manager:
        shared_cache = manager.dict()
        processes = [Process(target=cache_result, args=(shared_cache, f"item{i}", i*i)) for i in range(5)]

        for p in processes:
            p.start()
        for p in processes:
            p.join()

        print("Shared cache:", dict(shared_cache))
```

- `manager.dict()` gives us a shared dictionary.
- Every process writes to the same `cache` and it stays updated.

Let’s say your backend has several worker processes that perform expensive API calls or DB queries. You could use a shared cache dictionary to avoid redundant calls.

## Method 3: Using `multiprocessing.Lock` and `RLock`

**Why It Matters:**

Any shared resource accessed by multiple processes needs synchronization. `Lock` ensures only one process can use a critical section at a time.

**When to Use:**

Always use a lock when modifying shared state like `Value` or `Manager` objects.

```python
from multiprocessing import Process, Value, Lock

def safe_increment(counter, lock):
    for _ in range(1000):
        with lock:
            counter.value += 1

if __name__ == "__main__":
    counter = Value('i', 0)
    lock = Lock()
    processes = [Process(target=safe_increment, args=(counter, lock)) for _ in range(4)]

    for p in processes:
        p.start()
    for p in processes:
        p.join()

    print("Counter value:", counter.value)  # Should be 4000
```

## Advanced Usage: Shared Memory with `multiprocessing.shared_memory` (Python 3.8+)

This is for performance-critical tasks where you need to share large binary data, like numpy arrays.

```python
import numpy as np
from multiprocessing import Process, shared_memory

def modify_data(shm_name, shape):
    shm = shared_memory.SharedMemory(name=shm_name)
    np_array = np.ndarray(shape, dtype=np.int64, buffer=shm.buf)
    np_array += 10  # Modify in-place
    shm.close()

if __name__ == "__main__":
    data = np.array([1, 2, 3, 4], dtype=np.int64)
    shm = shared_memory.SharedMemory(create=True, size=data.nbytes)
    shared_array = np.ndarray(data.shape, dtype=data.dtype, buffer=shm.buf)
    shared_array[:] = data[:]

    p = Process(target=modify_data, args=(shm.name, data.shape))
    p.start()
    p.join()

    print("Modified array:", shared_array)  # [11, 12, 13, 14]
    shm.close()
    shm.unlink()
```

Think of an ML model pipeline where you preprocess large batches of data in parallel before feeding into a model. This helps avoid copying large arrays between processes.

## Caveats and Best Practices

- Always use locks when writing to shared state.
- `Manager()` is flexible but slower than `Value/Array` due to proxying.
- Avoid using too many processes writing to the same shared object. It can bottleneck performance.
- For performance-critical apps, use `shared_memory` or even switch to `joblib`, `ray`, or `dask`.

## Summary

| **Tool** | **Type** | **Best For** |
| --- | --- | --- |
| `Value`, `Array` | Shared memory | Simple values, numeric counters |
| `Manager.dict()` | Proxy object | Complex state like dicts, lists |
| `Lock`, `RLock` | Sync tool | Critical section safety |
| `shared_memory` | Raw memory | Large arrays, high performance needs |

## Final Thoughts

Managing shared state across processes isn’t as easy as it is with threads, but Python gives you the right tools. You just have to pick the right one for your use case. If you're building data-intensive pipelines, ML preprocessing stages, task queues, or API backends with multiple workers, understanding these shared-state mechanisms can make your code faster and more reliable.