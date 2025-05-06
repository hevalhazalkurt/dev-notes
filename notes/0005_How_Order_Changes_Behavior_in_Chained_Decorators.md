# How Order Changes Behavior in Chained Decorators

Check this post on my blog [here](https://hevalhazalkurt.com/blog/how-order-changes-behavior-in-chained-decorators/).

<br>

In the world of Python backend development, especially when building APIs with FastAPI, Flask, or Django, you’ll often work with function decorators to implement behaviors like authentication, caching, logging, or performance monitoring. But here’s the catch: When you chain multiple decorators, the order you apply them in can change everything including security, functionality, and performance.

This post walks you through:

- How decorator chaining works under the hood
- Why order matters (with real backend examples)
- Best practices to avoid nasty surprises

<br>

## A Little Refresher First: What’s a Decorator?

A decorator in Python is a function that takes another function and returns a modified version of it.

Example:

```python
def logger(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@logger
def get_data():
    return {"message": "Hello!"}
```

Calling `get_data()` will print a log before returning the result.

<br>

## Chaining Decorators = Wrapping Layers

Let’s say you apply multiple decorators to a function:

```python
@decorator_one
@decorator_two
def my_func():
    ...
```

Python translates this into:

```python
my_func = decorator_one(decorator_two(my_func))
```

In other words:

1. `decorator_two` wraps `my_func` (this happens first)
2. `decorator_one` wraps the result of that (this happens second)

But when you run `my_func()`, it executes in this order:

→ `decorator_one`

→ `decorator_two`

→ `my_func`

It’s last-applied, first-executed just like nested boxes.

<br>

## Real Backend Example: Auth + Logging

Let’s simulate a real backend scenario. Imagine you're working on an internal admin route:

```python
@log_request
@require_admin_auth
def get_admin_data():
    ...
```

Now let’s define our decorators.


```python
def require_admin_auth(func):
    def wrapper(*args, **kwargs):
        user = kwargs.get("user")
        if not user or not user.get("is_admin"):
            raise PermissionError("Unauthorized")
        return func(*args, **kwargs)
    return wrapper
```


```python
def log_request(func):
    def wrapper(*args, **kwargs):
        print(f"Request to {func.__name__} with args={args}, kwargs={kwargs}")
        return func(*args, **kwargs)
    return wrapper
```

<br>

### Route Simulation

```python
@log_request
@require_admin_auth
def get_admin_data(*args, **kwargs):
    return {"data": "Sensitive admin data"}
```

<br>

### Test Call

```python
get_admin_data(user={"username": "alice", "is_admin": False})
```

Output:

```
Request to wrapper with args=(), kwargs={'user': {'username': 'alice', 'is_admin': False}}
Traceback (most recent call last):
  ...
PermissionError: Unauthorized
```

Wait... it logged the request before checking auth! That could be a security concern, especially if the route leaks sensitive data (like full tokens, query args, etc).

<br>

## Change the Order

Let’s reverse the decorators:

```python
@require_admin_auth
@log_request
def get_admin_data(*args, **kwargs):
    return {"data": "Sensitive admin data"}
```

Now when you call it:

```python
get_admin_data(user={"username": "alice", "is_admin": False})
```

You get:

```
Traceback (most recent call last):
  ...
PermissionError: Unauthorized
```

No logging happens. Why? Because `require_admin_auth` runs first and blocks unauthenticated users before reaching `log_request`. Much better, more secure and faster too.

<br>

## More Examples of Why Order Matters

Let’s explore some decorator combos you might use in backend work:

### Example 1: `@cache` + `@authenticate`

```python
@authenticate
@cache
def get_user_data(user_id):
    ...
```

Wrong order! You could end up serving cached responses for other users. Better:

```python
@cache
@authenticate
def get_user_data(user_id):
    ...
```

Now:

1. First check if the request is authorized.
2. Then hit the cache.
3. Then go to the DB if needed.

<br>

### Example 2: `@retry` + `@log_failure`

```python
@log_failure
@retry(times=3)
def update_order():
    ...
```

This logs only the final failure, not all retries. If you want to log *every* retry failure:

```python
@retry(times=3)
@log_failure
def update_order():
    ...
```

Now the `log_failure` decorator gets applied *inside* each retry cycle.

<br>

## Debugging Tip: Add Traces

When you're unsure what's going on, add debug prints inside each decorator:

```python
def trace_decorator(name):
    def decorator(func):
        def wrapper(*args, **kwargs):
            print(f"[{name}] before {func.__name__}")
            result = func(*args, **kwargs)
            print(f"[{name}] after {func.__name__}")
            return result
        return wrapper
    return decorator

@trace_decorator("outer")
@trace_decorator("inner")
def process():
    print("...processing...")

process()
```

Output:

```
[outer] before wrapper
[inner] before process
...processing...
[inner] after process
[outer] after wrapper
```

Now you *see* the flow and how the decorators wrap each other.

<br>

## Best Practices for Chaining Decorators in Backend Code

- Put security decorators first (authentication, authorization)
- Add monitoring/logging last, don’t log unauthorized traffic
- Use `functools.wraps(func)` in custom decorators to preserve function metadata (critical for docs, testing, etc.)
- Use clear names for your decorators, chain order should read like a pipeline
- Avoid deeply nested wrappers, refactor if your function gets wrapped 5+ times