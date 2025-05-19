# Behind the Underscores EP09: Attribute Access (`__getattr__`, `__getattribute__`, `__setattr__`, `__delattr__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep09-attribute-access-__getattr__-__getattribute__-__setattr__-__delattr__/).

<br>

If you’ve been working with Python for a while, you’ve probably used objects and attributes all the time:

```python
class User:
    def __init__(self, name):
        self.name = name

u = User("Alice")
print(u.name)  # Accessing the 'name' attribute
```

Simple enough, right? But under the hood, Python gives you some powerful tools to customize what happens when you access, set, or delete attributes. These tools are special methods like `__getattr__`, `__getattribute__`, `__setattr__`, and `__delattr__`.

Let’s dive into what they are, what they do, and when and how to use them with real-world use cases.

## First: What Is an Attribute?

An attribute is just a variable that belongs to an object. When you write `obj.x`, `x` is the attribute. In classes, attributes are usually things like `name`, `email`, `age`, etc. You get or set them using dot notation.

## Attribute Access Internals

Python handles attribute access in this order:

1. Check the instance dictionary (`__dict__`)
2. Look in the class and its base classes
3. If not found, call `__getattr__` if it exists

But when you want to take control over how attribute access behaves, you can override four methods:

| **Method** | **When it Runs** | **Common Use Cases** |
| --- | --- | --- |
| `__getattribute__` | Always on attribute access | Logging, access control, wrappers |
| `__getattr__` | Only if attribute is missing | Lazy loading, proxies, fallbacks |
| `__setattr__` | On every attribute assignment | Validation, transformation, logging |
| `__delattr__` | On every attribute deletion | Protection, cleanup, auditing |

Let’s go through them one by one.

## `__getattribute__`: Called Every Time You Access an Attribute

```python
class Demo:
    def __getattribute__(self, name):
        print(f"Getting attribute: {name}")
        return super().__getattribute__(name)
```

This method is always called when you access any attribute on an instance. Even built-in ones like `__class__`.

Why use it?

- Logging or debugging attribute access
- Enforcing rules for access
- Adding dynamic behavior

If you override `__getattribute__`, you must call `super().__getattribute__(name)` inside it. Otherwise, you'll get a recursive loop and a `RecursionError`.

## `__getattr__`: Called Only If the Attribute Doesn't Exist

```python
class Lazy:
    def __getattr__(self, name):
        print(f"{name} not found. Creating it lazily.")
        return f"Default for {name}"
```

This method is called only when the attribute is missing. It’s great for:

- Providing defaults
- Lazy-loading values
- Building proxy/wrapper objects

```python
obj = Lazy()
print(obj.anything)  # "anything" doesn’t exist so __getattr__ is triggered
```

It will not run if the attribute already exists!

## `__setattr__`: Called When Setting an Attribute

```python
class Strict:
    def __setattr__(self, name, value):
        print(f"Setting {name} = {value}")
        super().__setattr__(name, value)

```

Use `__setattr__` when you want to:

- Validate or transform inputs
- Prevent or limit setting certain attributes
- Automatically log changes

```python
user = Strict()
user.age = 42  # Calls __setattr__
```

Just like with `__getattribute__`, you must call `super().__setattr__` or else the value won't be stored.

## `__delattr__`: Called When Deleting an Attribute

```python
class Guarded:
    def __delattr__(self, name):
        print(f"Attempting to delete {name}")
        if name == "id":
            raise AttributeError("You can't delete 'id'")
        super().__delattr__(name)
```

This method is useful when:

- You want to protect certain attributes from being deleted
- You want to log or audit deletions
- You need to keep cleanup logic centralized

```python
obj = Guarded()
obj.name = "temp"
del obj.name  # Calls __delattr__
```

## Example: Lazy Configuration Loader

Let’s say you want to load some configuration values only when they are needed:

```python
class Config:
    def __init__(self):
        self._store = {}

    def __getattr__(self, name):
        print(f"Loading config for {name}")
        value = f"default_{name}"
        self._store[name] = value
        return value

config = Config()
print(config.db_url)   # Loads lazily
print(config.api_key)  # Loads lazily
```

No `db_url` or `api_key` is defined beforehand. But thanks to `__getattr__`, they work anyway. Look at the output.

```
Loading config for db_url
default_db_url
Loading config for api_key
default_api_key
```

## Combine Methods for Power

You can combine these magic methods to create powerful behavior:

```python
class Magic:
    def __getattribute__(self, name):
        print(f"Accessing: {name}")
        return super().__getattribute__(name)

    def __getattr__(self, name):
        print(f"'{name}' not found. Using default.")
        return 42

    def __setattr__(self, name, value):
        print(f"Setting {name} = {value}")
        super().__setattr__(name, value)

    def __delattr__(self, name):
        print(f"Deleting {name}")
        super().__delattr__(name)
```

This kind of setup is great for:

- Wrapping APIs
- Building caching layers
- Creating domain-specific languages
- Validating models like in frameworks

## Final Thoughts

These methods may seem magical at first, but once you understand how they work, they open up a whole new level of control in your classes. Just remember:

- Always call `super()` inside these methods unless you're intentionally breaking behavior.
- Be cautious with `__getattribute__`, it’s very powerful and dangerous if misused.
- Use these tools to build smarter, more flexible, and maintainable code.