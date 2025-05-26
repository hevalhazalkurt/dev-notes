# Dataclasses vs Pydantic vs TypedDict vs NamedTuple in Python

Check this post on my blog [here](https://hevalhazalkurt.com/blog/dataclasses-vs-pydantic-vs-typeddict-vs-namedtuple-in-python/).

<br>

Python gives us a bunch of ways to model structured data: `dataclass`, `Pydantic`, `TypedDict`, and `NamedTuple`. But choosing the right one can be tricky, especially when you're building backend systems that need performance, clarity, and flexibility.

In this blog, we’ll explore these four tools in depth, not just what they are, but when you should or shouldn’t use them, with real-world backend development scenarios.

## 1. `dataclass`: The Pythonic Standard for Plain Data Objects

### The Basics

`dataclass` was introduced in Python 3.7 to make it easier to define classes used primarily for storing data. It removes the need for writing boilerplate code like `__init__`, `__repr__`, and `__eq__`.

```python
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
    email: str
```

Python auto-generates the constructor and other methods:

```python
user = User(1, "Alice", "alice@example.com")
print(user)  # User(id=1, name='Alice', email='alice@example.com')
```

### Use Case: In-Memory State Modeling

Use `dataclass` for modeling internal system state, especially when performance matters but you don’t need runtime validation.

Example → Caching a product catalog from the database

```python
@dataclass
class Product:
    id: int
    name: str
    price: float
    tags: list[str]
```

### Pros

- Built-in, fast, and lightweight
- Great for performance-critical code
- Customizable with `__post_init__`, `field()`, etc.

### Cons

- No built-in validation
- Weak error messages when input types are wrong
- Not designed for external API data

## 2. `Pydantic`: Validation-First Data Modeling

### The Basics

`pydantic` is a 3rd-party library focused on data validation and parsing. It uses Python type hints, just like `dataclass`, but adds a lot more power under the hood.

```python
from pydantic import BaseModel, EmailStr

class User(BaseModel):
    id: int
    name: str
    email: EmailStr
```

```python
user = User(id="1", name=123, email="test@example.com")
print(user)  # id=1 name='123' email='test@example.com'
```

### Use Case: API Input Validation

In a FastAPI backend, you'd typically use Pydantic models for parsing and validating incoming requests:

```python
from fastapi import FastAPI

app = FastAPI()

@app.post("/users")
def create_user(user: User):
    return user
```

### Pros

- Automatic type coercion (”1” → `1`)
- Strict validation of fields
- Great error messages
- JSON Schema and OpenAPI support

### Cons

- Slower than `dataclass`
- Extra dependency
- Slightly more overhead at runtime

## 3. `TypedDict`: Type Checking for Dicts

### The Basics

`TypedDict` from `typing` gives dictionaries structure, so your IDE and static analyzers can understand them.

```python
from typing import TypedDict

class UserDict(TypedDict):
    id: int
    name: str
    email: str
```

### Use Case: External JSON Contracts

Imagine you're consuming JSON from a third-party API:

```python
user_data: UserDict = {
    "id": 123,
    "name": "Bob",
    "email": "bob@example.com"
}
```

### Pros

- Works well with JSON data
- No runtime overhead
- Plays nicely with `mypy`

### Cons

- No runtime validation
- Just for type hints still a normal dict underneath

## 4. `NamedTuple`: Immutable, Indexed, and Typed

### The Basics

`NamedTuple` gives you the simplicity of a tuple but with named fields and type hints.

```python
from typing import NamedTuple

class User(NamedTuple):
    id: int
    name: str
    email: str
```

```python
user = User(1, "Alice", "alice@example.com")
print(user.name)  # Alice
```

### Use Case: Lightweight Configs and Constants

Use `NamedTuple` when you want immutable, structured data that behaves like a tuple for example, a read-only config or event signature.

### Pros

- Immutable and memory-efficient
- Works like a tuple
- Fully compatible with unpacking

### Cons

- No default values
- Verbose to update (have to create a new instance)
- Can feel limiting for complex data

## When to Choose What

| **Use Case** | **Best Choice** | **Why** |
| --- | --- | --- |
| Internal backend data models | `dataclass` | Fast, lightweight, Pythonic |
| API validation / user input | `Pydantic` | Rich validation and error reporting |
| Working with external JSON | `TypedDict` | Static type checking with dict flexibility |
| Immutable system events / configs | `NamedTuple` | Performance and immutability |

## Advanced Patterns

### Combining `dataclass` with `TypedDict`

Use `TypedDict` for external-facing inputs and `dataclass` for internal logic.

```python
class UserInput(TypedDict):
    id: int
    name: str
    email: str

@dataclass
class InternalUser:
    id: int
    name: str
    email: str
    is_active: bool = True
```

### `Pydantic.dataclasses`

Pydantic also supports a `dataclass` decorator that gives you Pydantic-like validation + dataclass syntax.

```python
from pydantic.dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
```

## What Not to Do

- Don’t use `TypedDict` expecting runtime checks.
- Don’t use `dataclass` to validate untrusted data from users.
- Don’t use `NamedTuple` for anything deeply nested or mutable.

## Final Thoughts

Each of these tools solves a specific problem. Think of them as different screwdrivers in your Python toolbox:

- Use `dataclass` when performance matters.
- Use `Pydantic` when you want safety and validation.
- Use `TypedDict` to talk to static type checkers.
- Use `NamedTuple` when you want tuples that are easier to read.

If you're building a modern backend system, chances are you'll end up using more than one of them, depending on the layer you're working in validation, business logic, storage, etc.