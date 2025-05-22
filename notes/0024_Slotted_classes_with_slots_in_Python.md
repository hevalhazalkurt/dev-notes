# Slotted classes with `__slots__ in Python

Check this post on my blog [here](https://hevalhazalkurt.com/blog/slotted-classes-with-__slots__-in-python/).

<br>

When we build backend systems in Python, performance isn't always our first concern. But sometimes, especially when you have thousands or even millions of objects in memory like API response models, simulation agents, or data processing pipelines, memory usage and speed start to matter. One of the lesser-known Python features that can help in such cases is `__slots__`.

In this blog, we'll explore `__slots__` from beginner level to more advanced use cases, with clear examples and backend-oriented use cases.

## What is `__slots__`?

In Python, every object by default stores its attributes in a special dictionary called `__dict__`. This dictionary allows us to dynamically add, remove, or modify attributes at runtime.

```python
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

u = User("Alice", "alice@example.com")
u.age = 30  # works just fine
```

But this flexibility comes at a cost. Every instance carries a dynamic dictionary, which adds memory overhead and makes attribute access a tiny bit slower. That's where `__slots__` comes in. `__slots__` lets you define a fixed set of attributes for a class. When used, Python doesn’t create a `__dict__` for each instance, reducing memory usage.

```python
class SlimUser:
    __slots__ = ['name', 'email']

    def __init__(self, name, email):
        self.name = name
        self.email = email
```

Now, trying to add a new attribute:

```python
s = SlimUser("Bob", "bob@example.com")
s.age = 25  # AttributeError: 'SlimUser' object has no attribute 'age'
```

## Why Should You Care?

In backend systems, we often work with many lightweight objects:

- API response models
- Background job data containers
- DTOs (Data Transfer Objects)
- Caching layers

In these cases, saving even a few bytes per object can scale into megabytes or more.

### Example: API Models

```python
class Product:
    def __init__(self, id, name, price):
        self.id = id
        self.name = name
        self.price = price
```

Suppose you're loading 1 million `Product` instances into memory for caching. Using `__slots__`:

```python
class SlimProduct:
    __slots__ = ['id', 'name', 'price']

    def __init__(self, id, name, price):
        self.id = id
        self.name = name
        self.price = price
```

Using `sys.getsizeof()` won't show the full memory benefit because it only shows shallow size, but if you compare `.__dict__` and `.__slots__` attributes or use tools like `pympler` or `tracemalloc` will show significant reduction when objects are deeply nested.

## How `__slots__` Works Under the Hood

When you use `__slots__`, Python internally creates a more static structure like a C struct instead of a dynamic dictionary. It uses descriptors and a slot table to manage attribute access.

Benefits:

- Reduced memory usage per object
- Slightly faster attribute access
- Prevents accidental addition of unexpected attributes

Drawbacks:

- No dynamic attribute assignment
- Can't easily use pickling without extra steps
- Subclassing can be tricky (see below)

## Combining `__slots__` and Inheritance

Slotted classes can't be freely combined like regular ones. If you subclass a slotted class, you need to define `__slots__` again even if it's empty.

```python
class Base:
    __slots__ = ['id']

class Child(Base):
    __slots__ = ['name']
```

If you forget to redefine `__slots__`, Python will revert to using `__dict__` in the subclass.

You can also include `__dict__` in the slots explicitly if you want partial flexibility:

```python
class PartiallyDynamic:
    __slots__ = ['id', '__dict__']
```

## `@dataclass` with `slots=True`

Python 3.10+ allows using `__slots__` automatically with `dataclasses`:

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Product:
    id: int
    name: str
    price: float
```

This is much cleaner and combines the benefits of dataclasses like auto-generated `__init__`, `__repr__`, etc. with memory savings.

## Caching API Responses

Suppose you're building a high-throughput API that returns lots of small JSON records from Redis or a database. Each record is parsed into a Python object.

```python
class CachedItem:
    __slots__ = ['id', 'category', 'value']

    def __init__(self, id, category, value):
        self.id = id
        self.category = category
        self.value = value
```

This keeps your memory usage lean, especially if you're storing thousands of these in memory. You can even combine `__slots__` with `__slots__ + property` to make fields read-only:

```python
class ImmutableItem:
    __slots__ = ['_id']

    def __init__(self, id):
        self._id = id

    @property
    def id(self):
        return self._id
```

## When Not to Use `__slots__`

Don’t use `__slots__` if:

- You rely on libraries that dynamically add attributes like some ORMs or serializers
- You need full pickling support
- You're writing code that heavily depends on `__dict__` introspection

It’s a tool, not a silver bullet. Use it when profiling shows memory usage is actually a concern.

## Summary

`__slots__` is a powerful but often overlooked feature in Python that can help reduce memory usage and improve attribute access performance in certain use cases. Especially in backend systems where you handle lots of structured but repetitive objects like response models or cached entities, slotted classes can provide measurable performance benefits.

Use them with care, test them in your architecture, and always validate with profiling tools before optimizing prematurely.