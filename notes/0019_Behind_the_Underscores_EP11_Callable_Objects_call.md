# Behind the Underscores EP11: Callable Objects: `__call__` 

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep11-callable-objects-__call__/).

<br>


When we think of calling something in Python, the first thing that comes to mind is usually a function. But Python gives us the power to go a step further: objects can behave like functions if they implement a special method called `__call__`. This concept might seem a bit strange at first, but once you understand how it works, you’ll find it incredibly useful, especially for building clean, modular backend systems.

In this blog post, we’ll dive deep into the `__call__` method. We'll explore:

- What `__call__` really does under the hood
- Why and when you should use it
- The potential pitfalls to watch out for
- Realistic backend examples

Let’s get started.

## What is the `__call__` Method?

In Python, every function is an object. And just like functions, any object that implements the `__call__` method can be used with parentheses to make it behave like a function:

```python
class Greeter:
    def __init__(self, name):
        self.name = name

    def __call__(self, greeting="Hello"):
        return f"{greeting}, {self.name}!"

hello_john = Greeter("John")
print(hello_john())           # Output: Hello, John!
print(hello_john("Hi"))       # Output: Hi, John!
```

Here, `hello_john` is an instance of `Greeter`, but thanks to `__call__`, we can treat it like a function.

## Why Would You Use `__call__`?

You might wonder: “Why not just use a regular function?”. That’s a fair question. Here are some reasons why using `__call__` can be a better choice in certain situations:

### 1. **Encapsulation of State**

Classes let you store data, or we say state, and behavior together. With `__call__`, you can package that logic in a neat callable form.

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor

    def __call__(self, number):
        return self.factor * number

triple = Multiplier(3)
print(triple(10))  # Output: 30
```

### 2. **Cleaner Dependency Injection**

In frameworks like FastAPI or Flask, you may want to inject behavior or data as a callable. Making your class instance callable makes the integration seamless.

### 3. **Custom Middleware or Pipelines**

If you're building your own middleware system, having callable objects makes it easier to write and chain operations.

## When **Not** to Use `__call__`

While `__call__` can be powerful, it can also lead to confusion or overengineering if used improperly.

- **Readability concerns**: Readers of your code might not expect that an instance behaves like a function. This can hurt code clarity.
- **Debugging difficulty**: Errors in a `__call__` implementation might be harder to track if your class hides too much logic inside it.
- **Unnecessary abstraction**: Sometimes a simple function is better. Don’t wrap everything in a class just because you can.

### Rule of Thumb

Use `__call__` when your object:

- Maintains state that’s useful across calls
- Needs to fit into APIs that expect callables
- Has a behavior that makes sense as a single primary action

## Backend Example #1: Request Logger Middleware

Imagine you’re building a custom logging middleware for a web server. You want to log the request method and path.

```python
class RequestLoggerMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] == "http":
            method = scope["method"]
            path = scope["path"]
            print(f"[LOG] {method} request to {path}")

        await self.app(scope, receive, send)
```

This can be used in a FastAPI or Starlette app like this:

```python
app.add_middleware(RequestLoggerMiddleware)
```

Because the middleware instance is callable, it fits perfectly into the ASGI lifecycle.

## Backend Example #2: Authorization Checker

Let’s say you need to check if a user has a specific role before allowing access to a route.

```python
class RoleChecker:
    def __init__(self, allowed_roles):
        self.allowed_roles = allowed_roles

    def __call__(self, user):
        if user.role not in self.allowed_roles:
            raise PermissionError("Access denied")
        return True

check_admin = RoleChecker(["admin"])

user = User(role="guest")
check_admin(user)  # Raises PermissionError
```

This is especially useful when you’re designing reusable validation or guard components.

## Summary: The Power of `__call__`

The `__call__` method turns your objects into callable powerhouses. It combines the simplicity of functions with the structure of classes, giving you the best of both worlds. But like any powerful tool, it should be used thoughtfully.
