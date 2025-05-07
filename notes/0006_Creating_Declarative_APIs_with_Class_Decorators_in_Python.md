# Creating Declarative APIs with Class Decorators in Python

Check this post on my blog [here](https://hevalhazalkurt.com/blog/creating-declarative-apis-with-class-decorators-in-python/).

<br>

When you use libraries like SQLAlchemy, Pydantic, or FastAPI, you might notice that you can declare behavior just by writing classes with certain decorators. This style is called declarative programming. You describe what you want, and the system figures out how to do it. One of the ways this magic happens in Python is through class decorators.

In this blog post, we’ll walk through what class decorators are, how they can be used to build declarative APIs, and build a simple example from scratch to make the idea clear.

<br>

## What is a Class Decorator?

Let’s start with a quick refresher. A class decorator is a function that takes a class object as input and returns a possibly modified or wrapped class.

```python
def my_decorator(cls):
    print(f"Decorating class: {cls.__name__}")
    return cls

@my_decorator
class MyClass:
    pass
```

When Python sees `@my_decorator`, it runs `my_decorator(MyClass)`. This is very similar to function decorators, but the input is a class.

<br>

## What’s a Declarative API?

A **declarative API** lets the developer express intent with very little code, often without explicitly invoking logic. For example:

```python
@route("/hello")
class HelloWorld:
    def get(self):
        return "Hello, world!"
```

Here, `@route("/hello")` might register this class as a handler for a web request. The user doesn't need to know how routing works, just declare the intent.

<br>

## Building a Mini Declarative System

Let’s build a simple example where we can declare configuration for services using class decorators. Imagine we want to register all services that provide some functionality and access their metadata later.

### Step 1: The Registry

We need a place to keep track of services:

```python
service_registry = {}
```

<br>

### Step 2: The Class Decorator

Now, let’s make a decorator that registers any class with a name and optional config:

```python
def service(name=None, **config):
    def decorator(cls):
        service_name = name or cls.__name__
        service_registry[service_name] = {
            "class": cls,
            "config": config
        }
        return cls
    return decorator
```

<br>

### Step 3: Declaring Services

```python
@service(name="email", retries=3, timeout=5)
class EmailService:
    def send(self, message):
        print(f"Sending email: {message}")

@service(name="sms", provider="Twilio")
class SMSService:
    def send(self, message):
        print(f"Sending SMS: {message}")
```

Now, we’ve created a *declarative* way to define services. All metadata is stored automatically:

```python
from pprint import pprint
pprint(service_registry)
```

Output:

```bash
{
  'email': {
    'class': <class '__main__.EmailService'>,
    'config': {'retries': 3, 'timeout': 5}
  },
  'sms': {
    'class': <class '__main__.SMSService'>,
    'config': {'provider': 'Twilio'}
  }
}
```

<br>

## What Did We Just Do?

- We created a **registry** for services.
- We used a **class decorator** to capture metadata and register classes.
- Developers can now declare new services just by decorating a class.

They don’t need to know anything about how the registry works, it’s declarative.

<br>

## Taking It Further

You can do much more with class decorators:

- Automatically inject configuration into `__init__`
- Wrap or replace class methods
- Add new class-level behavior dynamically

For example, to inject config into a class:

```python
def service(name=None, **config):
    def decorator(cls):
        class Wrapped(cls):
            def __init__(self, *args, **kwargs):
                super().__init__(*args, **kwargs)
                self.config = config
        service_name = name or cls.__name__
        service_registry[service_name] = {
            "class": Wrapped,
            "config": config
        }
        return Wrapped
    return decorator
```

Now you can do:

```python
email = EmailService()
print(email.config)  # {'retries': 3, 'timeout': 5}
```

<br>

## Real-World Examples in Frameworks

Many popular Python frameworks use this pattern:

- **FastAPI** uses decorators like `@app.get()` to register route handlers declaratively.
- **Pydantic** uses `@dataclass` or model classes to declare schemas and validation rules.
- **SQLAlchemy** uses declarative base classes to define ORM models.

Once you understand class decorators, you start to see the pattern behind these intuitive APIs.

<br>

## Class Decorators vs Metaclasses

Class decorators are great for simple behavior injection and registration. They:

- Are easier to write and understand
- Don't require touching inheritance chains
- Can be stacked or reused

Metaclasses are more powerful, but:

- Can be overkill for many cases
- Are harder to debug
- Require more boilerplate

Use class decorators when you want to keep things readable, and reach for metaclasses when you need low-level control over class creation itself.

<br>

## Adding a Utility: Service Instantiation

You can also add a helper function to instantiate registered services:

```python
def get_service(name):
    cls = service_registry[name]['class']
    return cls()

email_service = get_service("email")
email_service.send("Test message")
```

This makes the system not only declarative but also easy to use dynamically.

<br>

## Writing Tests for Your Decorators

Here’s a simple unit test for the registry logic:

```python
def test_service_registry():
    assert "email" in service_registry
    service_info = service_registry["email"]
    assert service_info["config"]["retries"] == 3
```

Or test the injected config:

```python
def test_email_service_config():
    service = EmailService()
    assert service.config["timeout"] == 5
```

Tests help ensure your decorators behave as expected when used in larger systems.

<br>

## Common Pitfalls to Watch Out For

- **Losing metadata**: If you wrap the class, the name and docstring may change. Preserve them manually if needed.
- **Mutating shared state**: If your decorator changes global state (like a registry), be sure it's thread-safe.
- **Returning the wrong object**: Always return the new or original class from the decorator.
- **Forgetting `super().__init__()`**: When wrapping classes, don't forget to call the original constructor.

<br>

## Final Thoughts

Class decorators are a powerful tool in Python, especially when designing APIs that feel intuitive and expressive. They let you shift from writing code that says do this to code that says this is what I want.

If you’re building tools, frameworks, or even internal SDKs, consider how class decorators might help you build clean, declarative APIs that your future self and your teammates will thank you for.