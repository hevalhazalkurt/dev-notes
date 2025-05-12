# Behind the Underscores EP02: Object Initialization and Construction Methods (`__new__`, `__init__`, `__del__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep02-object-initialization-and-construction-methods-__new__-__init__/).

<br>

In Python, object creation is controlled through a special set of methods: `__new__`, `__init__`, and `__del__`. These methods give us the flexibility to customize how objects are created, initialized, and destroyed. While `__init__` is the most commonly used method, the other two, `__new__` and `__del__`, allow for even more control, especially in advanced patterns such as singleton patterns, object pooling, and memory management.

In this article, we'll explore:

- What `__new__`, `__init__`, and `__del__` are and how they work.
- Why and when to override them.
- Best practices and potential pitfalls.
- Real-world examples demonstrating how and when to use these methods.

## Conceptual Overview

Before diving into the examples, it's crucial to understand the basic roles and behaviors of each method:

### `__new__(cls, *args, **kwargs)`

- **Responsible for**: Object creation.
- **What happens**: The `__new__` method is responsible for creating and returning a new instance of the class. It's called first, before `__init__`. If the object is immutable, `__new__` has to return the fully initialized object, as the state of the object cannot be modified after creation.
- **When to override**: `__new__` is usually overridden in advanced use cases, such as implementing design patterns like Singleton, Flyweight, subclassing immutable types like `str`, `tuple`, etc., or managing object creation efficiently like object pooling.
- **Common use cases**: Singleton pattern, Flyweight pattern, subclassing immutable types, object pooling (caching), metaclass manipulation.

### `__init__(self, *args, **kwargs)`

- **Responsible for**: Initializing the object's state after creation.
- **What happens**: `__init__` is called immediately after the object is created by `__new__`. Here, we initialize instance attributes and perform any necessary setup for the object.
- **When to override**: Most of the time, we override `__init__` to initialize instance attributes, validate input, or perform some other setup. It’s the method that most developers interact with.
- **Common use cases**: Initializing object attributes, validating input, performing setup operations, dependency injection, and configuration parsing.

### `__del__(self)`

- **Responsible for**: Cleaning up the object before it’s destroyed.
- **What happens**: The `__del__` method is called when an object’s reference count drops to zero, indicating that the object is about to be garbage collected. It is intended for cleanup tasks, such as releasing external resources like file handles, network sockets.
- **When to override**: You override `__del__` if you need to release external resources when the object is destroyed. However, it should be used with caution due to Python's non-deterministic garbage collection system.
- **Common use cases**: Closing files, releasing network connections, cleaning up database connections, and other resource management tasks.

## Deep Dive into `__new__`

While most developers rarely need to override `__new__`, it provides powerful capabilities for advanced object creation strategies. Here are some scenarios where `__new__` shines.

### 2.1 Immutable Subclasses

Python’s built-in immutable types like `str`, `tuple`, and `frozenset` cannot have their internal state changed once they are created. Therefore, any transformation on these types must happen at the moment of creation. That’s where `__new__` comes into play. 

Let’s look at that example: 

```python
class UpperStr(str):
    def __new__(cls, content):
        # Modify the content before the instance is created
        instance = super().__new__(cls, content.upper())
        return instance

s = UpperStr("hello")
print(s)  # Output: HELLO
```

In this example:

- The class `UpperStr` inherits from `str` which is immutable.
- The `__new__` method is overridden to modify the string by converting it to uppercase before creating the object.
- Since `str` objects are immutable, the transformation must occur during creation, not after.

This approach ensures that once the object is created, it cannot be modified, preserving the immutability of the `str` type.

### 2.2 Singleton Pattern

The Singleton design pattern ensures that a class has only one instance throughout the application. To achieve this, we use `__new__` to control object creation and ensure that only one instance is ever created.

```python
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

In this example:

- The `__new__` method checks whether the class has already created an instance.
- If the instance doesn't exist, it creates one using the `super().__new__(cls)` call.
- Subsequent attempts to create instances of the class will return the same object, enforcing the singleton behavior.

### 2.3 Object Pooling / Flyweight Pattern

The Flyweight pattern reduces memory usage by sharing objects that have the same state. Instead of creating a new instance every time, objects are reused from a shared pool.

```python
class Flyweight:
    _cache = {}

    def __new__(cls, name):
        if name not in cls._cache:
            cls._cache[name] = super().__new__(cls)
        return cls._cache[name]

# Only one instance for each unique 'name'
obj1 = Flyweight("apple")
obj2 = Flyweight("apple")
obj3 = Flyweight("banana")

print(obj1 is obj2)  # Output: True (same instance)
print(obj1 is obj3)  # Output: False (different instance)
```

In this example:

- The `Flyweight` class caches instances in a dictionary based on the `name` passed to `__new__`.
- If an instance with the same `name` already exists, the existing instance is returned; otherwise, a new one is created.
- This reduces memory usage when dealing with a large number of objects with the same state.

## Mastering `__init__`

The `__init__` method is essential for initializing objects. While `__new__` controls object creation, `__init__` is where we initialize the object's state. Here are a few advanced use cases of `__init__`.

### 3.1 Dependency Injection

Dependency Injection (DI) is a technique where an object’s dependencies are provided (injected) to it during its initialization. This decouples the object from its dependencies, making it easier to test and modify.

```python
class DatabaseConnection:
    def connect(self):
        return "Connected to the database."

class Service:
    def __init__(self, db_connection):
        self.db = db_connection

    def perform_task(self):
        return f"Task performed using {self.db.connect()}"

db = DatabaseConnection()
service = Service(db)

print(service.perform_task())  
# Output: Task performed using Connected to the database.
```

In this example:

- The `Service` class requires a `DatabaseConnection` object to function.
- Instead of creating the `DatabaseConnection` inside the `Service` class, it’s injected during initialization.
- This pattern makes the code more flexible and easier to test, as you can inject mock dependencies during unit tests.

### 3.2 Configuration Parsing

In many applications, we need to parse a configuration file or dictionary and initialize an object with the values. This can be easily handled by overriding `__init__`.

```python
class Config:
    def __init__(self, config_dict):
        for k, v in config_dict.items():
            setattr(self, k, v)

config_dict = {"host": "localhost", "port": 8080}
config = Config(config_dict)

print(config.host)  # Output: localhost
print(config.port)  # Output: 8080
```

In this example:

- The `Config` class takes a dictionary and dynamically assigns each key-value pair as an attribute of the object using `setattr()`.
- This pattern is useful when the attributes are not known ahead of time, such as in configuration management systems.

### 3.3 Runtime Type Checking

You can use `__init__` to enforce type checks or validation on the input arguments. This is useful for ensuring that the object is always initialized with valid data.

```python
class User:
    def __init__(self, age):
        if not isinstance(age, int):
            raise TypeError("Age must be an integer.")
        self.age = age

user = User(25)            # Works fine
user_invalid = User("25")  # Raises TypeError: Age must be an integer.
```

In this example:

- The `User` class ensures that the `age` argument passed to `__init__` is an integer.
- If it's not an integer, a `TypeError` is raised, which prevents invalid data from being assigned to the object.

## Subtleties of `__del__`

The `__del__` method is Python’s destructor, called when an object is about to be garbage collected. However, it comes with caveats that should be carefully considered.

### 4.1 Basic Example

Here’s a simple example of using `__del__` to clean up external resources such as file handles or network sockets:

```python
class FileWriter:
    def __init__(self, path):
        self.file = open(path, 'w')

    def __del__(self):
        print("Closing file.")
        self.file.close()
```

In this example:

- The `FileWriter` class opens a file in `__init__`.
- The `__del__` method ensures that the file is closed when the object is garbage collected.

### 4.2 Caveats and Best Practices

- **Non-deterministic behavior**: The garbage collector is non-deterministic, meaning that `__del__` may not always be called when you expect.
- **Reference cycles**: If objects refer to each other in a cycle like object A refers to object B, and object B refers to object A, `__del__` may not be called.
- **Exceptions in `__del__`**: If an exception is raised in `__del__`, it will be ignored, and you may not even be aware that there was an issue.

### Use Context Managers Instead

For managing resources like file handles, it's better to use context managers (`with` statement) instead of relying on `__del__`.

```python
class FileWriter:
    def __enter__(self):
        self.file = open("file.txt", "w")
        return self.file

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file.close()

with FileWriter() as file:
    file.write("Hello, world!")
```

In this example:

- The `FileWriter` class implements the context manager protocol (`__enter__` and `__exit__`).
- The `with` statement ensures that the file is automatically closed when the block exits, even if an exception is raised.

## Custom Metaclasses and `__new__`

Metaclasses are classes that define the behavior of other classes. `__new__` in metaclasses is used to customize class creation itself.

```python
class Meta(type):
    def __new__(cls, name, bases, dct):
        dct['created_by'] = "Meta"
        return super().__new__(cls, name, bases, dct)

class MyClass(metaclass=Meta):
    pass

obj = MyClass()
print(obj.created_by)  # Output: Meta
```

In this example:

- A custom metaclass `Meta` is defined, and its `__new__` method adds a new attribute `created_by` to any class that uses it.
- `MyClass` uses `Meta` as its metaclass, and the `created_by` attribute is automatically added to the class.

## When to Use What?

| **Use case** | **Method to override** | **Why?** |
| --- | --- | --- |
| Subclassing `str`, `tuple`, etc. | `__new__` | Immutable objects need early state setup |
| Enforcing Singleton/Flyweight | `__new__` | Control over object creation and reuse |
| Basic object state initialization | `__init__` | Assign values, validate input, inject deps |
| Releasing external resources | `__del__` (rare) | Clean-up logic at end of object lifecycle |
| Safe resource management | `__enter__`/`__exit__` | Prefer over `__del__` for determinism |

By mastering these special methods, you gain precise control over object lifecycles, enable more efficient memory and resource usage, and lay the foundation for clean, extensible software architecture in Python.