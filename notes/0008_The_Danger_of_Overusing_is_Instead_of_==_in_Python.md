# The Danger of Overusing is Instead of == in Python

Check this post on my blog [here](https://hevalhazalkurt.com/blog/the-danger-of-overusing-is-instead-of-in-python/).

<br>

If you've been working with Python for a while, you've probably used both `is` and `==` in your code. They look similar, they even read similarly. But underneath the hood, they do very different things. And using `is` when you meant to use `==` can lead to subtle, nasty bugs that are incredibly hard to track down.

## What's the Difference Between `is` and `==`?

- `==` checks if two values are equal.
- `is` checks if two variables point to the same object in memory.

Here’s a simple way to remember it:

> 👉 == → “Do these things look the same?”
> 
> 
> 👉 `is` → “Are these things *literally the same object*?”
> 

**A Quick Example**

```python
a = [1, 2, 3]
b = [1, 2, 3]

print(a == b)  # True: the contents are the same
print(a is b)  # False: different objects in memory
```

Even though `a` and `b` contain the same list values, they’re **not** the same object. Python created two separate lists.

## Where Things Get Dangerous

The trouble starts when developers assume `is` behaves like `==`, or vice versa, especially because sometimes `is` works "accidentally" due to Python's internal optimizations.

**Example 1: Strings**

```python
x = "hello"
y = "hello"

print(x == y)  # True
print(x is y)  # True? ...sometimes
```

This can be `True` for `is`, because Python “interns” (reuses) short strings to save memory. But watch this:

```python
a = "hello world! this is a very long string"
b = "hello world! this is a very long string"

print(a == b)  # True
print(a is b)  # False
```

Suddenly `is` doesn’t work, even though the strings are equal. Why? Because Python didn’t intern the longer string. Relying on `is` here gives you flaky, inconsistent results depending on factors you don't control like string length or Python implementation.

### Example 2: Integers

Python also caches small integers between `-5` and `256`.

```python
x = 256
y = 256

print(x is y)  # True

a = 257
b = 257

print(a is b)  # False
```

It’s easy to write code that “passes” some tests but fails in production. And these bugs can be near-impossible to debug later.

## So When Should You Use `is`?

There are valid uses for `is`. But they’re specific and rare.

### Use `is` When Checking for Singleton Objects

For example:

```python
if my_var is None:
    ...
```

This is the correct way to check for `None`, because `None` is a singleton. There’s only one `None` object in Python.

Other valid cases:

- `my_obj is True`
- `my_obj is False`

But even for `True` and `False`, `==` is usually safer in data-heavy code (e.g. pandas, NumPy), because those libraries redefine what truthy means.

## Real-World Bug Scenarios

Here are a few real bugs I've seen:

### Bug 1: Wrong Comparison in Loops

```python
for item in my_list:
    if item is "done":  # oops
        break
```

This might work for some strings, but fail silently for others. The correct check is:

```python
for item in my_list:
    if item == "done":
        break
```

### Bug 2: False Negatives in Data Validation

```python
user_input = input("Type yes or no: ")

if user_input is "yes":  # might fail even if input was 'yes'
    print("Confirmed!")
```

You might never know this failed unless your user types `"yes"` and nothing happens.

### Bug 3: Unreliable Conditional Logic

```python
x = 1000
y = 10 * 100

if x is y:  # False! Even though values are the same
    do_something()
```

This will silently not run `do_something()` even though most humans would say “1000 is equal to 1000.”

## TL;DR - When to Use Which?

| **Operation** | **Use `==`?** | **Use `is`?** |
| --- | --- | --- |
| Value equality | ✅ | ❌ |
| Object identity | ❌ | ✅ |
| Compare to `None` | ❌ | ✅ |
| Compare to constant (e.g., string, int) | ✅ | ❌ |
| Writing reliable, portable code | ✅ | ❌ (except for `None`) |

## Final Advice

- Stick with `==` unless you're 100% sure you're checking for identity.
- Use `is` only when checking for `None`, or comparing against sentinel objects.
- Don’t rely on Python’s internal optimizations like interning or small int caching. They’re implementation details and can change between versions or platforms.