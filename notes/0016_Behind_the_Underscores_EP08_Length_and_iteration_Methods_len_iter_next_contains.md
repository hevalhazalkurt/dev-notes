# Behind the Underscores EP08: Length and iteration Methods (`__len__`, `__iter__`, `__next__`, `__contains__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep08-length-and-iteration-methods-__len__-__iter__-__next__-__contains__/).

<br>

If you've ever used a `for` loop, checked if a value is in a list, or called `len()` on something, you've been using Python's data model. Specifically, special methods like `__len__`, `__iter__`, `__next__`, and `__contains__`. These “dunder methods” let you make your own Python objects behave like built-in types. Want your custom class to work with `len()`, `in`, or a `for` loop? These methods are the key.

Let’s break them down with examples, everyday explanations.

## 1. `__len__`: Make `len()` Work on Your Object

### What it does:

When you call `len(something)`, Python looks for a `__len__()` method under the hood.

### Why it matters:

If you build a class and want `len(my_obj)` to return something meaningful, like how many items are inside it, you define `__len__`.

### Example:

```python
class ShoppingCart:
    def __init__(self):
        self.items = []

    def add_item(self, item):
        self.items.append(item)

    def __len__(self):
        return len(self.items)

cart = ShoppingCart()
cart.add_item("apple")
cart.add_item("banana")

print(len(cart))  # Output: 2
```

## 2. `__iter__`: Make Your Object Iterable

### What it does:

This method lets Python know how to start looping over your object.

### Why it matters:

With `__iter__`, your object can be used in a `for` loop, list comprehensions, or anything that expects something iterable.

### Example:

```python
class Numbers:
    def __init__(self):
        self.data = [10, 20, 30]

    def __iter__(self):
        return iter(self.data)  # Could also return a custom iterator

nums = Numbers()
for n in nums:
    print(n)  # 10 20 30
```

## 3. `__next__`: Define What Happens On Each Iteration Step

### What it does:

This method returns the next item from your object. It’s usually paired with `__iter__`.

### Why it matters:

If you're building a custom iterator from scratch, `__next__` is where the actual “step-by-step" happens.

### Example:

```python
class Countdown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self  # The object is its own iterator

    def __next__(self):
        if self.current < 0:
            raise StopIteration
        val = self.current
        self.current -= 1
        return val

for num in Countdown(3):
    print(num)  # 3 2 1 0
```

## 4. `__contains__`: Control How `in` Works

### What it does:

This method is triggered when you write something like:

```python
if "apple" in cart:
```

### Why it matters:

It lets you define what it means for an item to “be inside” your object.

### Example:

```python
class ShoppingCart:
    def __init__(self):
        self.items = ["apple", "banana"]

    def __contains__(self, item):
        return item in self.items

cart = ShoppingCart()
print("banana" in cart)  # True
print("milk" in cart)    # False
```

If you don’t implement `__contains__`, Python will automatically fall back to looping over the object using `__iter__`. That’s helpful!

## How These Work Together

Let’s look at a bigger picture. When you do this:

```python
if "something" in my_obj:
```

Python tries the methods in this order:

1. Try `__contains__`.
2. If not available, try to iterate with `__iter__` and compare each item.
3. If neither is available, it raises a `TypeError`.

When you do this:

```python
for x in my_obj:
```

Python does this behind the scenes:

1. Calls `__iter__()` to get an iterator.
2. Repeatedly calls `__next__()` on the iterator.
3. Stops when `StopIteration` is raised.

## Real-World Use Cases

- **Custom collections**: Build your own data containers that behave like lists, sets, or queues.
- **Data wrappers**: Wrap API or database results in classes that are iterable and length-aware.
- **Game states or UI systems**: Control which components are active using `__contains__`.

## You Don’t Always Need All of Them

You might just need `__iter__` if you're building a class that behaves like a list.

You might only need `__len__` if you're showing count in a UI or enforcing limits.

And sometimes, using `yield` and generators is even easier for iteration than managing `__next__` manually.

## Summary

| **Method** | **Purpose** | **Used by...** |
| --- | --- | --- |
| `__len__` | Define what `len(obj)` returns | `len()` |
| `__iter__` | Make object iterable | `for`, `in`, list comps |
| `__next__` | Get the next item in a loop | `next()`, `for` |
| `__contains__` | Define `in` behavior | `"x" in obj` |

## Wrapping Up

These special methods are like magic doors into Python’s built-in behavior. Once you learn to use them, you can make your classes feel like real native Python types. If you're building something reusable or preparing for interviews, mastering `__len__`, `__iter__`, `__next__`, and `__contains__` is a great move.