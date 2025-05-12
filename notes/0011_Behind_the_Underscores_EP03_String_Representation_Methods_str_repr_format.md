# Behind the Underscores EP03: String Representation Methods (`__str__`, `__repr__`, `__format__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep03-string-representation-methods-__str__-__repr__-__format__/).

<br>

If you've ever worked with Python and printed an object only to see something like `<MyClass object at 0x102b4a310>`, you're not alone. That kind of output is the default behavior when Python doesn't know how to turn your object into a meaningful string. Most people shrug it off. But here’s the thing:

> Python’s string representation methods (`__str__`, `__repr__`, and `__format__`) are low-effort, high-impact tools that can drastically improve the quality of your code, especially for debugging, logging, and building user-facing tools.
> 

Let’s break down what they are, why they matter, and how to actually use them right.

## What Are These Methods, Really?

Python has three core string conversion hooks in classes:

| **Method** | **What It's For** | **Called When...** |
| --- | --- | --- |
| `__str__` | For humans (readable) | `print(obj)` or `str(obj)` |
| `__repr__` | For devs (unambiguous, debug) | Shell/REPL display, logging, containers |
| `__format__` | For custom formatting logic | `format(obj)`, `f"{obj:spec}"` |

Let’s go beyond the basics and explore where each of these really shows up and how to master them.

## `__str__`: A Friendly Face for Your Object

This is what users see when you print the object.

```python
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def __str__(self):
        return f"{self.name} <{self.email}>"

user = User("Alice", "alice@example.com")
print(user)

# Output: Alice <alice@example.com>
```

That looks much better than `<User object at 0xABC123>`.

**Integrates With f-strings**

```python
user = User("Bob", "bob@example.com")
print(f"User: {user}")  # Uses __str__ automatically
```

If `__str__` isn't defined, Python falls back to `__repr__`. If that's not defined either, you get the ugly memory address thing.

## `__repr__`: For Debugging and Developers

This should return a developer-readable version of the object. Ideally, something that could recreate the object if passed to `eval()` when possible.

```python
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def __repr__(self):
        return f"User(name={self.name!r}, email={self.email!r})"

print([User("Alice", "a@example.com"), User("Bob", "b@example.com")])

# Output: [User(name='Alice', email='a@example.com'), User(name='Bob', email='b@example.com')]
```

Notice the use of `!r` . It's shorthand for using `repr()` inside f-strings.

### Why `__repr__` Is Crucial:

- Logging systems use it
- Lists and dicts use it for elements
- Your REPL/shell relies on it
- Debuggers and tracebacks show it

## `__format__`: Your Object on the Runway

This is the lesser-known star of the show. It's used when formatting with `str.format()` or f-strings with format specifiers.

```python
class Price:
    def __init__(self, amount):
        self.amount = amount

    def __format__(self, spec):
        if spec == "euro":
            return f"\u20ac{self.amount:.2f}"
        elif spec == "usd":
            return f"${self.amount:.2f}"
        return f"{self.amount:.2f}"

p = Price(19.99)
print(f"Price: {p:euro}")  # €19.99
print(f"Price: {p:usd}")   # $19.99
```

### Tip:

Combine `__format__` with `locale` for internationalized output. It also works great in reporting tools or APIs where you want different string views.

## Mistakes to Avoid

### Returning Non-Strings

```python
def __str__(self):
    return 123  # TypeError!
```

Always return strings, not numbers or None.

### Making `__str__` and `__repr__` Identical

They have different jobs. Don’t make them twins unless your object is dead simple.

### Calling `str(self)` Inside `__str__`

```python
def __str__(self):
    return str(self)  # Infinite recursion!
```

Use `self.attribute` instead.

## Real-World Places Where These Matter

### 1. Logging and Debugging in a Web App (Using `__repr__`)

**Scenario**: You're building a Django or FastAPI backend and want better logging for your `UserSession` objects.

```python
class UserSession:
    def __init__(self, user_id, ip_address, active):
        self.user_id = user_id
        self.ip_address = ip_address
        self.active = active

    def __repr__(self):
        return (f"UserSession(user_id={self.user_id!r}, "
                f"ip_address={self.ip_address!r}, active={self.active})")
```

**Why it matters**:

```python
session = UserSession(42, "192.168.0.1", True)

# Logs will show this:
print(session)

# Output: UserSession(user_id=42, ip_address='192.168.0.1', active=True)
```

This avoids ambiguity and shows the dev-friendly internal state, useful for bug tracing.

### Displaying Clean Info in a CLI App (Using `__str__`)

**Scenario**: You’re building a CLI tool that lists files or reports.

```python
class Report:
    def __init__(self, name, status):
        self.name = name
        self.status = status

    def __str__(self):
        return f"[{self.status.upper()}] {self.name}"
```

**Usage**:

```python
report = Report("Q2 Financial Summary", "ok")
print(report)

# Output: [OK] Q2 Financial Summary
```

Clear, readable output for end-users. If the user doesn’t care about internals, `__str__` hides them elegantly.

### Multi-Currency Display in a Finance App (Using `__format__`)

**Scenario**: You're building a financial dashboard where amounts need to be shown in various currencies.

```python
class Money:
    def __init__(self, amount):
        self.amount = amount

    def __format__(self, spec):
        if spec == 'usd':
            return f"${self.amount:,.2f}"
        elif spec == 'eur':
            return f"€{self.amount:,.2f}"
        elif spec == 'btc':
            return f"{self.amount:.6f} BTC"
        return f"{self.amount:.2f}"  # Default format
```

**Usage**:

```python
m = Money(15432.75)
print(f"Price: {m:usd}")  # Output: Price: $15,432.75
print(f"Price: {m:eur}")  # Output: Price: €15,432.75
print(f"Price: {m:btc}")  # Output: Price: 15432.750000 BTC
```

Dynamic formatting lets you use the same object in multiple UI contexts, based on format specifiers.

### Cleaner Test Failures with `__repr__`

**Scenario**: You're writing tests using `pytest` or `unittest`.

```python
class Product:
    def __init__(self, id, name):
        self.id = id
        self.name = name

    def __repr__(self):
        return f"Product(id={self.id!r}, name={self.name!r})"
```

**Test output**:

```python
assert Product(1, "Banana") == Product(2, "Apple")

# Output from test framework:
# E       AssertionError: assert Product(id=1, name='Banana') == Product(id=2, name='Apple')
```

Without `__repr__`, you’d just see memory addresses — which helps no one.

### Use in Pandas or Jupyter for Previewing Data Objects

**Scenario**: You’re building custom objects to analyze tabular data and use them inside Jupyter notebooks.

```python
class DataPoint:
    def __init__(self, label, value):
        self.label = label
        self.value = value

    def __repr__(self):
        return f"<{self.label}: {self.value}>"
```

```python
data = [DataPoint("Temperature", 21.5), DataPoint("Humidity", 60)]
data  # In Jupyter you'll see a list with readable reprs
```

Without this, Jupyter just shows a raw list of object memory locations.

### Interactive HTML Representation (Jupyter)

```python
class HTMLUser:
    def __init__(self, name, role):
        self.name = name
        self.role = role

    def _repr_html_(self):
        return f\"\"\"<b>{self.name}</b> - <i>{self.role}</i>\"\"\"
```

`_repr_html_()` isn’t technically part of `__str__`/`__repr__`, but it's closely related. It allows Jupyter to display rich previews.

### Expert Tip: Use a Repr Mixin

Create a reusable `ReprMixin` to auto-generate a good `__repr__`:

```python
class ReprMixin:
    def __repr__(self):
        attrs = ', '.join(f"{k}={v!r}" for k, v in self.__dict__.items())
        return f"{self.__class__.__name__}({attrs})"

class Product(ReprMixin):
    def __init__(self, id, title):
        self.id = id
        self.title = title

print(Product(10, "Banana"))
# Output: Product(id=10, title='Banana')
```


## Conclusion

String representation methods aren’t just fluff. They are your objects' public voice. They control how readable, debuggable, and intuitive your system is.

Whether you’re building a library, a web API, a machine learning pipeline, or a CLI tool, spending a few extra minutes designing your `__str__`, `__repr__`, and `__format__` methods will pay off in clarity, ease of debugging, and polish.

- Use `__str__` for people
- Use `__repr__` for devs
- Use `__format__` for customization

And never underestimate their power again.