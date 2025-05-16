# Behind the Underscores EP07: Container protocol (`__getitem__`, `__setitem__`, `__delitem__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep07-container-protocol-__getitem__-__setitem__-__delitem__/).

<br>

Have you ever used square brackets in Python? Of course you have:

```python
my_list = [1, 2, 3]
print(my_list[0])  # prints 1

my_dict = {"name": "Alice"}
print(my_dict["name"])  # prints "Alice"
```

Behind the scenes, Python calls some special methods to make this happen. These methods are part of what’s called the container protocol and the three big players are:

- `__getitem__` – for accessing items
- `__setitem__` – for assigning values
- `__delitem__` – for deleting items

These special methods are what allow your objects to act like containers just like lists, dictionaries, or even NumPy arrays. 

In this post, we’ll break down each of these, show you how they work, and walk through real-world examples to help make it stick.

## So What Is the Container Protocol?

Python allows objects to behave like containers, something that can hold, access, or remove values using square brackets, if you implement certain magic methods in your class.

| **Method** | **Purpose** | **Triggered by...** |
| --- | --- | --- |
| `__getitem__` | Accessing a value by key or index | `obj[key]` |
| `__setitem__` | Assigning a value | `obj[key] = value` |
| `__delitem__` | Deleting a key or index | `del obj[key]` |

## Let's Build One from Scratch

Let’s start with a simple custom class that mimics a dictionary.

```python
class MyContainer:
    def __init__(self):
        self._data = {}

    def __getitem__(self, key):
        print(f"Getting item for key: {key}")
        return self._data[key]

    def __setitem__(self, key, value):
        print(f"Setting item: {key} = {value}")
        self._data[key] = value

    def __delitem__(self, key):
        print(f"Deleting item for key: {key}")
        del self._data[key]
```

Now we can use this class just like a dictionary:

```python
box = MyContainer()
box["fruit"] = "apple"      # Setting item
print(box["fruit"])         # Getting item
del box["fruit"]            # Deleting item
```

Output:

```
Setting item: fruit = apple
Getting item for key: fruit
apple
Deleting item for key: fruit
```

Now your custom object now supports square bracket operations.

## Deep Dive into Each Method

### 1. `__getitem__(self, key)`

This method is triggered when you access an item like `obj[key]`. The syntax is like below:

```python
def __getitem__(self, key):
    return self._data[key]
```

- You can also support slicing, like `obj[1:4]`, by checking if the `key` is a `slice` object.
- Great for building lists, matrices, caches, etc.

Let’s look at a little example.

```python
class MyList:
    def __init__(self, data):
        self.data = data

    def __getitem__(self, index):
        return self.data[index]

my_list = MyList([10, 20, 30])
print(my_list[1])  # Output: 20
```

Here, `my_list[1]` works just like a normal list because we told Python how to handle it.

If you want to support slicing too, you can implement it like that:

```python
class MyList:
    def __init__(self, data):
        self.data = data

    def __getitem__(self, key):
    if isinstance(key, slice):
        return self.data[key.start:key.stop:key.step]
    else:
        return self.data[key]
        

my_list = MyList([10, 20, 30, 40, 50])
print(my_list[1:3])  # Output: [20, 30]
```

**Other example use-cases:**

- Lazy-loading data
- Paginating API responses
- Custom file readers

### 2. `__setitem__(self, key, value)`

Called when you do `obj[key] = value`. First, let’s look at its syntax.

```python
def __setitem__(self, key, value):
    self._data[key] = value
```

You can use this to:

- Add validation like allowing only integers
- Transform input like always storing uppercase strings
- Enforce key limits like a cache

Look at that example.

```python
class LoggingDict:
    def __init__(self):
        self._data = {}

    def __setitem__(self, key, value):
        print(f"Setting {key} to {value}")
        self._data[key] = value

    def __getitem__(self, key):
        return self._data[key]

ld = LoggingDict()
ld["a"] = 123     # prints: Setting a to 123
print(ld["a"])    # prints: 123

```

You can use this to validate inputs, format data, or even reject unwanted keys.

### 3. `__delitem__(self, key)`

Called when you do `del obj[key]`.

```python
def __delitem__(self, key):
    del self._data[key]
```

Use cases:

- Cleaning up memory
- Logging deletions
- Automatically updating linked resources

If we want creating a tracking class. Then we can implement these methods like below. 

```python
class TrackingDict:
    def __init__(self):
        self._data = {}

    def __setitem__(self, key, value):
        self._data[key] = value

    def __getitem__(self, key):
        return self._data[key]

    def __delitem__(self, key):
        print(f"Deleting key: {key}")
        del self._data[key]

td = TrackingDict()
td["x"] = 5
del td["x"]  # prints: Deleting key: x
```

## Why You Should Care

- Frameworks like Django and Flask use these methods to customize how objects behave.
- APIs and libraries often override these to simplify access (e.g., objects behaving like dicts).
- Cleaner code — your custom objects can be more intuitive to use.

## Let's Build a Smarter Dictionary

Before we wrap up, let’s put all the theory into action with a real-world example. Imagine you're building a configuration system for your application, something that holds settings like `"DEBUG"`, `"TIMEOUT"`, or `"HOST"`. You want to make sure only specific keys are allowed, that values are the right type like `str` or `bool`, and that small mistakes like capitalization errors or invalid types don’t silently break your app.

This is where container protocol methods come in handy.

In the example below, we’ll build a custom class called `StrictConfigStore`. It behaves like a dictionary but with extra powers: it validates keys, enforces value types, treats keys case-insensitively, and even logs every time you get, set, or delete a value. This is the kind of tool you'd use in real projects where reliability and safety matter.

Let’s dive in!

```python
class StrictConfigStore:
    def __init__(self, allowed_keys: list[str], value_type: type):
        self._store = {}
        self._allowed_keys = {key.lower() for key in allowed_keys}
        self._value_type = value_type

    def _normalize_key(self, key):
        if not isinstance(key, str):
            raise TypeError("Key must be a string.")
        key = key.lower()
        if key not in self._allowed_keys:
            raise KeyError(f"'{key}' is not a valid configuration key.")
        return key

    def __getitem__(self, key):
        key = self._normalize_key(key)
        print(f"Accessing '{key}'...")
        return self._store[key]

    def __setitem__(self, key, value):
        key = self._normalize_key(key)
        if not isinstance(value, self._value_type):
            raise ValueError(
                f"Invalid value type: expected {self._value_type.__name__}, got {type(value).__name__}"
            )
        print(f"Setting '{key}' = {value}")
        self._store[key] = value

    def __delitem__(self, key):
        key = self._normalize_key(key)
        if key not in self._store:
            raise KeyError(f"Key '{key}' not set.")
        print(f"Deleting '{key}'")
        del self._store[key]

    def __repr__(self):
        return f"<StrictConfigStore {self._store}>"
```

So what it does?

- Key normalization:
    - All keys are case-insensitive and stored in lowercase.
    - `"DEBUG"` and `"debug"` are treated the same.
- Type enforcement:
    - Only values of the correct type like `bool`, `int`, `str` are accepted.
- Validation on access and delete:
    - Prevents silent errors due to typos or wrong types.
- Custom error messages:
    - Much more informative than native Python errors.
- Prints on access/modify/delete:
    - Helps you track operations for debugging.

Now, let’s use it.

```python
store = StrictConfigStore(allowed_keys=["debug", "timeout", "host"], value_type=str)

store["DEBUG"] = "true"           # OK (case-insensitive)
store["timeout"] = "30"           # OK
print(store["host"])              # KeyError: 'host' not set yet

store["timeout"] = 30             # ValueError: expected str, got int

store["host"] = "localhost"
print(store["host"])              # prints: "localhost"

del store["debug"]                # OK
del store["debug"]                # KeyError: Key 'debug' not set.
```

This type of structure is useful in situations where:

- You want to enforce rules around what keys/values are allowed
- You need case-insensitive keys common in headers or configs
- You’re working in a team setting and want to prevent silent bugs from typos or wrong types
- You want to easily plug logging into every data operation

## Wrapping Up

Python’s container protocol gives you magical powers to make your objects behave like dictionaries, lists, or anything in between. Mastering `__getitem__`, `__setitem__`, and `__delitem__` can make your classes more flexible, powerful, and fun to use.