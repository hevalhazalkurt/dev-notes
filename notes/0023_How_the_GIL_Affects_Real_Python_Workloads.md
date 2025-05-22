# How the GIL Affects Real Python Workloads

Check this post on my blog [here](https://hevalhazalkurt.com/blog/how-the-gil-affects-real-python-workloads/).

<br>

If you’ve ever tried to speed up your Python application by adding threads, only to see... nothing change. Well, welcome to the world of the Global Interpreter Lock (GIL). In this post, we’re going to break down how the GIL affects concurrency performance, especially in real-life backend workloads.

## What’s the GIL?

The Global Interpreter Lock is a mutex that prevents multiple native threads from executing Python bytecodes at once in the CPython interpreter. It’s like a nightclub bouncer: only one thread can enter the Python bytecode dancefloor at a time even if there’s space for more. This means that:

- Threads in Python aren’t truly concurrent when doing CPU work.
- You don’t get free performance gains just by spawning threads for heavy computation.
- I/O-heavy code can still benefit from threading, because the GIL is released during I/O.

## CPU-Bound vs I/O-Bound: Why It Matters

Let’s define these first:

- CPU-bound: Code that mainly uses CPU cycles like data crunching, image processing.
- I/O-bound: Code that mostly waits like network requests, database calls, file I/O.

Why does this matter? Because the GIL hurts you most when your code is CPU-bound. That’s when threads fight over the lock. But for I/O-bound code, threads take turns nicely, because the GIL is released during blocking I/O.

## Example #1: CPU-Bound Task (With Threads)

Let’s simulate a CPU-heavy workload using threads:

```python
import threading
import time

COUNT = 50_000_000

def cpu_heavy():
    x = 0
    for _ in range(COUNT):
        x += 1

start = time.time()

thread1 = threading.Thread(target=cpu_heavy)
thread2 = threading.Thread(target=cpu_heavy)

thread1.start()
thread2.start()
thread1.join()
thread2.join()

print(f"Threads (CPU-bound): {time.time() - start:.2f}s")
```

What you’ll see: The two threads don’t make it faster. It may even take longer than running one after another.

Why? They’re both fighting for the GIL. Only one can do actual Python work at a time.

## Example #2: CPU-Bound Task (With Multiprocessing)

Now let’s try the same workload using `multiprocessing`:

```python
from multiprocessing import Process
import time

COUNT = 50_000_000

def cpu_heavy():
    x = 0
    for _ in range(COUNT):
        x += 1

start = time.time()

p1 = Process(target=cpu_heavy)
p2 = Process(target=cpu_heavy)

p1.start()
p2.start()
p1.join()
p2.join()

print(f"Processes (CPU-bound): {time.time() - start:.2f}s")
```

Now it’s much faster! Each process gets its own GIL and memory space — so they can run in parallel across CPU cores.

## Example #3: I/O-Bound Task (With Threads)

Let’s simulate a real backend workload — calling a slow API:

```python
import threading
import time
import requests

URL = "https://httpbin.org/delay/2"  # waits 2 seconds before replying

def fetch():
    response = requests.get(URL)
    print(response.status_code)

start = time.time()

threads = [threading.Thread(target=fetch) for _ in range(5)]

for t in threads:
    t.start()
for t in threads:
    t.join()

print(f"Threads (I/O-bound): {time.time() - start:.2f}s")
```

Even though each request takes 2 seconds, the whole program finishes in ~2 seconds, not 10.

Why? Because `requests.get()` blocks on I/O and releases the GIL, allowing other threads to run.

## Threads Work Well With I/O

In backend APIs, you often:

- Call other APIs
- Talk to databases
- Read files

These are all I/O-bound, so threading can actually help even with the GIL. For example, a simple FastAPI endpoint like:

```python
@app.get("/status")
def get_status():
    requests.get("https://a-service.com/ping")
    return {"status": "ok"}
```

Can scale better under concurrent users if served by a thread-based ASGI server like Uvicorn with workers.

## asyncio vs multiprocessing vs threading

| **Task Type** | **Best Tool** | **Why** |
| --- | --- | --- |
| CPU-bound | multiprocessing | Avoids the GIL by using processes |
| I/O-bound | threading / asyncio | GIL released during I/O |
| Mixed | Split workloads | Use threads for I/O, processes for CPU |

## Summary

- Python threads won’t help you speed up CPU-heavy code because of the GIL.
- For CPU-bound work, use `multiprocessing`, or call out to C extensions that release the GIL.
- For I/O-heavy work, Python threads (or `asyncio`) are great.
- Benchmark your own code before optimizing. You might be blaming the wrong thing.