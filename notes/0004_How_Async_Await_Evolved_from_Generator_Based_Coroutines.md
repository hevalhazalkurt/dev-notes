# How Async/Await Evolved from Generator-Based Coroutines

Check this post on my blog [here](https://hevalhazalkurt.com/blog/how-asyncawait-evolved-from-generator-based-coroutines/).

<br>

### A simple, friendly deep dive into Python's async journey

When you see code like this in Python:

```python
async def main():
    await fetch_data()
```

…it feels modern, sleek, and intuitive. But Python didn’t always make asynchronous programming this smooth. In fact, before `async` and `await`, async code was kind of a mess, full of workarounds, clever tricks, and a lot of *“Wait, how does this even work?”* moments.

At the heart of it all was an underrated feature: **generators**. Believe it or not, today’s async/await syntax is the result of years of building on the humble `yield`.

Let’s walk through this evolution with examples along the way.

<br>

## Step 1: It All Started with `yield`

You’ve probably used a generator before. Generators let a function pause partway through and come back later. That pause happens thanks to the `yield` keyword.

Here’s a classic example:

```python
def countdown(n):
    while n > 0:
        yield n
        n -= 1

for num in countdown(3):
    print(num)
```

This prints:

```
3
2
1
```

Instead of calculating everything up front, a generator *lazily* gives you the next value only when you ask for it—like a vending machine that gives you one snack at a time.

<br>

## Step 2: You Can *Send* Values Into Generators, Really?

Here’s where things get interesting. People realized you could not only get values out of a generator, but also send values in. Look at this example:

```python
def echo():
    while True:
        received = yield
        print(f"You said: {received}")

e = echo()
next(e)           # Start the generator
e.send("Hello!")  # "You said: Hello!"
```

Mind blown, right? `yield` is no longer just a way to output values, it becomes a two-way street. This idea opened the door to a new use case: **coroutines**. These are special functions that pause, wait for input, do something, and then pause again. You can use them to coordinate things like I/O, message handling, or step-by-step pipelines.

<br>

## Step 3: Using Coroutines to Do Real Work

Let’s say we want to create a coroutine that processes messages, like a mini chatbot:

```python
def chatbot():
    name = yield "Hi there! What’s your name?"
    yield f"Nice to meet you, {name}!"

bot = chatbot()
print(next(bot))         # "Hi there! What’s your name?"
print(bot.send("Alice")) # "Nice to meet you, Alice!"
```

This feels conversational. The coroutine waits for your input (`yield`), does something with it, then continues. Neat! But… it still feels manual. What if we want to run lots of these coroutines, switching between them automatically? That’s when people started building **event loops**.

<br>

## Step 4: The Rise of Event Loops (and a Little Chaos)

As the use of coroutines grew, Python developers began creating event loops—systems that would keep track of multiple coroutines and jump between them when each was ready to run. Early on, this was all done using **generator-based coroutines** and `yield`.

Example using an old pattern:

```python
def task():
    print("Starting task")
    yield  # Pretend we wait for I/O here
    print("Task resumed")

t = task()
next(t)  # Output: "Starting task"
next(t)  # Output: "Task resumed"
```

Libraries like **Tornado** and **Twisted** used generators and event loops to simulate async behavior, long before Python had any `async` syntax. But the code got messy. Managing when to `next()`, when to `send()`, how to pass results around, and how to handle exceptions… it was a headache.

<br>

## Step 5: Enter `yield from` — Chaining Generators

To make things smoother, Python 3.3 introduced `yield from` (for more detail you can check t[his post](https://hevalhazalkurt.com/blog/the-power-of-yield-from-in-python-generators/)). Let’s say you had two generators, and one wanted to “delegate” to the other:

```python
def child():
    yield 1
    yield 2

def parent():
    yield from child()
    yield 3

for val in parent():
    print(val)
```

Output:

```
1
2
3
```

`yield from` made it easier to chain generators together. It also made coroutines easier to compose. Suddenly, your event loop code became a little less ugly. At this point, things were getting better, but still not ideal.

<br>

## Step 6: `asyncio` and the Precursor to Async/Await

Python 3.4 introduced the `asyncio` module and gave us:

- An official **event loop**
- The `@asyncio.coroutine` decorator
- A formal way to `yield from` coroutines

It looked like this:

```python
import asyncio

@asyncio.coroutine
def main():
    yield from asyncio.sleep(1)
    print("Done!")

loop = asyncio.get_event_loop()
loop.run_until_complete(main())
```

It worked… but it was weird-looking. `@asyncio.coroutine`? `yield from`? Not exactly beginner-friendly. The Python community wanted something cleaner. And that led to the final leap.

<br>

## Step 7: `async` and `await`, At Last!

In Python 3.5, two new keywords were added: `async` and `await`.

Instead of writing:

```python
@asyncio.coroutine
def fetch():
    yield from asyncio.sleep(1)
```

You could now write:

```python
async def fetch():
    await asyncio.sleep(1)
```

It’s clearer, cleaner, and just… makes more sense. Now, any function defined with `async def` is a **coroutine function**. And inside it, you can use `await` to pause and wait for other async operations to finish.

<br>

## But Under the Hood… It’s Still Generators

This is the wild part: even though you're writing `async def`, Python is still using some of the same machinery that powered generators. Here’s what happens when you call an `async def` function:

```python
async def say_hi():
    return "hi"

coroutine = say_hi()
print(coroutine)  # <coroutine object say_hi at 0x...>
```

That object is a coroutine. You can’t use `next()` or `.send()` on it like a generator, but the idea is similar. It represents a paused function that you can resume later.

The key difference: coroutine objects are now first-class citizens in Python, and tools like `asyncio` or `trio` know exactly how to work with them.

<br>

## So…

The evolution from generators to async/await is one of the coolest examples of how Python gradually improves without breaking everything. Instead of jumping straight to some shiny new feature, Python took small, thoughtful steps:

- It started with a simple tool (`yield`)
- Saw how developers used it in creative ways
- Added improvements (`send`, `yield from`)
- Then made it *official* with `async` and `await`

Understanding this journey doesn’t just help you appreciate async/await more—it also helps you debug tricky async code and understand older codebases.

So next time you're `await`ing something in your app, give a little nod to `yield`. It walked so `async` could run.