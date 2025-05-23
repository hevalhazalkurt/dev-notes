# The Art of Scope Management in Modular Python Design

Check this post on my blog [here](https://hevalhazalkurt.com/blog/the-art-of-scope-management-in-modular-python-design/).

<br>


When you work on a large Python codebase, especially in backend projects using Django, FastAPI, or Flask, you probably see the chaos that poor scope management can cause. From mysterious bugs and unpredictable state to namespace collisions and tangled dependencies, things get messy fast when variable scope isn’t handled with care.

## What Is “Scope” in Python?

In simple terms, scope is where a variable can be seen or used. For example:

```python
def greet():
    name = "Alice"
    print(name)

print(name)  # NameError: name is not defined
```

Here, `name` is only visible inside the `greet()` function. That’s its scope. Python uses something called the LEGB rule to decide how it looks for variables.

## The LEGB Rule: Python’s Scope Lookup Chain

This rule stands for:

- **L**ocal – variables defined inside a function.
- **E**nclosing – variables in parent functions when functions are nested.
- **G**lobal – variables defined at the module level.
- **B**uilt-in – stuff that comes with Python, like `len`, `print`, `range`.

When you reference a variable, Python starts at the innermost scope and moves outward until it finds it. Here’s a quick example:

```python
x = "global"

def outer():
    x = "enclosing"

    def inner():
        x = "local"
        print(x)

    inner()

outer()  # prints "local"
```

If you remove `x = "local"` from `inner()`, Python prints `"enclosing"` — and if that’s gone too, it prints `"global"`. This rule is simple… until your app grows.

## Why Scope Matters

Let’s say you're building a backend service with FastAPI, and you start breaking your code into modules:

```
/project
  ├── main.py
  ├── database.py
  ├── models.py
  ├── routers/
  │     └── user.py
```

If you don’t manage scope carefully, you’ll run into things like:

- Circular imports
- Unpredictable globals
- Variables that vanish or leak
- Hard-to-debug state in production

Let’s see how you can manage scope cleanly.

## Rule 1: Keep Your Global Scope Clean

Your `main.py` is your entry point. It should only:

- Start the app
- Include global configuration (maybe via `os.environ`)
- Register routers and services

**Good:**

```python
# main.py
from fastapi import FastAPI
from routers import user

app = FastAPI()

app.include_router(user.router)
```

**Bad:**

```python
# main.py
db_connection = connect_to_db()
SOME_MAGIC_GLOBAL_STATE = {}

# Used all over your app without structure
```

Why it’s bad: When you use global mutable objects, things can go wrong in concurrency, testing, or scaling. They also make your app harder to test.

**Instead**: Pass state as arguments or use dependency injection, FastAPI supports this.

## Use Module Scope for Reusability

Imagine you have a `database.py` file that sets up your DB engine:

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine("sqlite:///example.db")
SessionLocal = sessionmaker(bind=engine)
```

This is good use of module scope. When you import `SessionLocal`, it’s consistent and controlled.

Example:

```python
# routers/user.py
from fastapi import Depends
from sqlalchemy.orm import Session
from database import SessionLocal

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.get("/users/")
def read_users(db: Session = Depends(get_db)):
    return db.query(User).all()
```

Notice how we don’t expose too much. We don’t let `engine` float around everywhere. `SessionLocal` is the scoped, reusable object.

## Avoid Import-Time Side Effects

A common mistake:

```python
# models.py
from database import SessionLocal

SessionLocal().execute("DROP TABLE users;")  # this runs on import
```

Importing a module should not perform dangerous actions. That’s a scope + timing issue.

Instead:

- Keep logic inside functions.
- Only run them when explicitly called.
- Avoid code at the top level that mutates or acts.

## Advanced Scope Patterns for Large Projects

Let’s get into deeper waters.

### 1. Dependency Injection with Scope Control

FastAPI lets you use function-level scope to inject services:

```python
def get_current_user(token: str = Depends(oauth2_scheme)):
    user = decode_token(token)
    return user
```

This is better than making `current_user` a global variable. It’s safer, more testable, and better scoped.

### Using Classes to Encapsulate State

Sometimes, you need state. Don’t abuse globals, use classes:

```python
# services/user_service.py
class UserService:
    def __init__(self, db):
        self.db = db

    def get_user(self, user_id):
        return self.db.query(User).filter_by(id=user_id).first()
```

In your endpoint:

```python
@router.get("/users/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    service = UserService(db)
    return service.get_user(user_id)
```

Here, `db` is passed down cleanly, no surprises, no globals.

### Factory Functions and Closures for Configurable Behavior

Sometimes closures help with scope:

```python
def make_greeting(prefix):
    def greet(name):
        return f"{prefix}, {name}!"
    return greet

hello = make_greeting("Hello")
print(hello("Alice"))  # Hello, Alice!
```

Use this in backends to build things like custom validators, filters, or pipelines with stored context.

## Clean Scope = Clean Code

To wrap it up, good scope management makes your Python code:

- Easier to test
- Easier to maintain
- Safer in production
- Faster to understand

Here’s a quick cheat sheet:

| **Do This** | **Avoid This** |
| --- | --- |
| Use local variables inside funcs | Using global variables as shared state |
| Pass arguments explicitly | Relying on outer scope invisibly |
| Encapsulate state with classes | Spreading config across files |
| Keep module top-level clean | Running side effects on import |