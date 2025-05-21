# Behind the Underscores EP13: Metaprogramming Methods (`__class__`, `__bases__`, `__mro__`, `__instancecheck__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep13-metaprogramming-__class__-__bases__-__mro__-__instancecheck__/).

<br>


Metaprogramming in Python is like programming about programming. It means writing code that can change how other code behaves. Sounds deep? It is. But it's also powerful. In this post, we'll break down some of the most important metaprogramming methods in Python. These include `__class__`, `__bases__`, `__mro__`, `__instancecheck__`, and `__subclasshook__`. We’ll keep things practical and show how they might be useful in real backend development scenarios.

## First, What Is a Metaclass?

A metaclass is simply the class of a class. Just like objects are created from classes, classes themselves are created from metaclasses.

Think of it this way:

- 🧱 You use a class to build objects.
- 🛠️ Python uses a metaclass to build classes.

By default, Python uses a built-in metaclass called `type`, but you can define your own to control how classes behave when they’re created.

Here’s a super simple example:

```python
class MyMeta(type):
    def __new__(cls, name, bases, dct):
        print(f\"Creating class: {name}\")
        return super().__new__(cls, name, bases, dct)

class MyClass(metaclass=MyMeta):
    pass

# Output: Creating class: MyClass
```

As you can see, `MyMeta` intercepted the moment `MyClass` was being defined and injected its own logic. That’s the core idea.

## What’s Python’s Default Metaclass Structure?

Like I mentioned before, in Python, the default metaclass is `type`. Every class you define is actually an instance of `type`, unless you say otherwise.

```python
class A:
    pass

print(type(A))  # <class 'type'>
```

Even built-in classes follow this structure:

```python
print(type(int))     # <class 'type'>
print(type(object))  # <class 'type'>
```

So what does `type` actually do? It:

- Creates new classes when you define them
- Handles the `__mro__`, `__bases__`, and other class-level mechanics
- Serves as the parent of all metaclasses

Unless you explicitly specify another metaclass, Python will always fall back to `type`. This is what makes metaprogramming possible in the first place. You’re building on top of Python’s default behavior and customizing it to your needs.

Now, let’s look at how we can make them custom for our codes. 

## 1. `__class__`: Who Are You Really?

The `__class__` attribute tells you the class of an instance. It’s a simple yet powerful way to inspect objects at runtime.

### Example: Debugging API Payloads

Imagine you're working on a Flask or FastAPI backend. You receive an object that should be a `UserPayload`, but you want to double-check.

```python
class UserPayload:
    pass

data = UserPayload()
print(data.__class__)  # <class '__main__.UserPayload'>
```

### Real-life use

Let’s say you log incoming data types for debugging:

```python
def log_type(obj):
    print(f"Received object of type: {obj.__class__.__name__}")
```

## 2. `__bases__`: Know Your Parents

This attribute gives you the immediate parent classes of a class. It’s like checking a family tree.

### Example: Plugin System

Say you’re building a plugin system and you want to validate that a plugin inherits from `BasePlugin`:

```python
class BasePlugin:
    pass

class MyPlugin(BasePlugin):
    pass

print(MyPlugin.__bases__)  # (<class '__main__.BasePlugin'>,)
```

### Real-life use

When auto-discovering classes for registration:

```python
if BasePlugin in MyPlugin.__bases__:
    register_plugin(MyPlugin)
```

## 3. `__mro__`: Method Resolution Order

This is how Python decides which method to call when there are multiple inheritance paths. You can inspect it to understand how your code will behave.

### Example: Service Layer Conflict Resolution

In a layered service architecture, if two parent classes implement the same method, `__mro__` helps resolve ambiguity.

```python
class AuthService:
    def execute(self):
        print("Auth logic")

class LoggingService:
    def execute(self):
        print("Logging logic")

class UserService(AuthService, LoggingService):
    pass

print([cls.__name__ for cls in UserService.__mro__])
# ['UserService', 'AuthService', 'LoggingService', 'object']
```

Python will call `AuthService.execute()` first.

## 4. `__instancecheck__`: Redefining `isinstance()`

You can override how `isinstance()` works by using a metaclass and implementing `__instancecheck__`.

### Example: Duck Typing Microservices

In microservices, you often care about behavior, not type. Suppose anything with a `.run()` method is a valid Job:

```python
class JobMeta(type):
    def __instancecheck__(cls, instance):
        return callable(getattr(instance, 'run', None))

class Job(metaclass=JobMeta):
    pass

class EmailJob:
    def run(self):
        print("Sending email")

print(isinstance(EmailJob(), Job))  # True
```

Even though `EmailJob` doesn’t inherit from `Job`, it’s considered an instance.

## 5. `__subclasshook__`: Virtual Subclasses

This is used when you want `issubclass()` to return True even when there is no inheritance as long as the class behaves like it should.

### Example: Abstract Base Classes in a Backend

Let’s define a repository interface that all storage backends should implement:

```python
from abc import ABCMeta

class Repository(metaclass=ABCMeta):
    @classmethod
    def __subclasshook__(cls, subclass):
        return (hasattr(subclass, 'save') and
                hasattr(subclass, 'delete'))

class SQLRepository:
    def save(self): pass
    def delete(self): pass

print(issubclass(SQLRepository, Repository))  # True
```

This is great when you want to enforce contracts by behavior rather than strict inheritance.

## When Do We Actually Need Metaprogramming?

Most of the time, plain classes and functions will get you pretty far in Python. But metaprogramming becomes useful when your code needs to be more dynamic, self-aware, or extendable.

Here are some situations where metaprogramming methods shine:

- **Framework Design:** If you're building your own framework like Django or FastAPI, you'll often need to hook into how classes are created or validated.
- **Plugin Systems:** Want to load plugins automatically just by defining new classes? `__subclasshook__` and `__bases__` can help.
- **Custom Validation Rules:** You might want to check if objects follow certain rules without enforcing inheritance which is great for loose coupling.
- **Duck Typing by Behavior:** Instead of checking inheritance, you check if an object behaves a certain way like having a `.run()` method.
- **Debugging or Logging:** Use `__class__`, `__mro__`, and others to inspect what kind of object you're dealing with at runtime.

In short, when you want to make Python smarter about how it sees and uses classes or object, this is when metaprogramming helps.

## When Should We Be Careful Using Them?

Just because you can use metaprogramming doesn’t mean you always should. These tools are powerful, but they come with sharp edges.

Here’s when you should take a step back:

- **Readability Drops:** Overusing metaclasses and custom hooks can make code really hard to follow for other developers and even future you.
- **Debugging Becomes Harder:** If `isinstance()` or `issubclass()` doesn’t behave normally, that can cause confusion and bugs that are tough to trace.
- **Performance Overhead:** Dynamically checking attributes in `__instancecheck__` or `__subclasshook__` might slow things down if used on large datasets or high-traffic systems.
- **Third-Party Conflicts:** Some libraries or tools may expect default behaviors, so customizing too much might break integrations.

Use them like spices: a pinch here and there can make your code elegant. Overdo it, and you’ll ruin the dish.

## Final Thoughts

Metaprogramming methods like `__class__`, `__bases__`, `__mro__`, `__instancecheck__`, and `__subclasshook__` are not just abstract ideas. They give you fine-grained control over how your code behaves, especially in large-scale backend systems where flexibility, extensibility, and loose coupling are essential.

Use them wisely, and you’ll unlock the ability to write smarter, more adaptive Python code.