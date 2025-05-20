# Behind the Underscores EP12: Descriptor Protocol (`__get__`, `__set__`, `__delete__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep12-descriptor-protocol-__get__-__set__-__delete__/).

<br>

Python is full of powerful features that let you write clean, reusable, and elegant code. One of these features, often misunderstood or underused, is the descriptor protocol. Descriptors let you hook into attribute access and build flexible, reusable behaviors around it. They're used heavily behind the scenes in properties, class methods, static methods, and even Django ORM models.

In this blog post, we’ll break down the descriptor protocol special methods: `__get__`, `__set__`, and `__delete__`, and see how they’re useful in backend development.

## What Is a Descriptor?

A descriptor is simply a class that implements any of the following methods:

- `__get__(self, instance, owner)`
- `__set__(self, instance, value)`
- `__delete__(self, instance)`

Once a descriptor class is assigned as a class attribute, it controls access to that attribute. This allows you to customize what happens when someone gets, sets, or deletes an attribute from an instance of the owner class.

## The Three Special Methods

### 1. `__get__`: Controls how an attribute is read

```python
def __get__(self, instance, owner):
    # Return the value to be used when the attribute is accessed
```

- `instance`: the instance accessing the attribute
- `owner`: the class of the instance

### 2. `__set__`: Controls how an attribute is written

```python
def __set__(self, instance, value):
    # Handle setting a new value to the attribute
```

- `value`: the value being assigned to the attribute

### 3. `__delete__`: Controls what happens when an attribute is deleted

```python
def __delete__(self, instance):
    # Handle deletion of the attribute
```

Now, let’s explore them in a few real-life backend examples.

## Example 1: Using a Descriptor in a Product Tag System

Imagine you’re building a backend for an e-commerce site. Each product can have tags, and these tags store various metadata like value, shape, and name. Internally, this data lives in a `data` object, but you want to make access clean and consistent on the main `Tag` class.

```python
class TagDataDescriptor:
    def __init__(self, attr):
        self.attr = attr

    def __get__(self, tag, owner):
        return getattr(tag.data, self.attr)

    def __set__(self, tag, value):
        setattr(tag.data, self.attr, value)

class Tag:
    value = TagDataDescriptor('value')
    shape = TagDataDescriptor('shape')
    name = TagDataDescriptor('name')

    def __init__(self, data):
        self.data = data

class TagData:
    def __init__(self, value, shape, name):
        self.value = value
        self.shape = shape
        self.name = name

# Usage
raw_data = TagData("sale", "circle", "Discount")
tag = Tag(raw_data)

print(tag.value)   # Prints: sale
tag.shape = "square"
print(tag.data.shape)  # Prints: square
```

### So, What’s Happening

- When you do `tag.value`, the descriptor’s `__get__` method is triggered.
- It fetches the actual data from `tag.data.value`.
- When you set `tag.shape = "square"`, the descriptor forwards it to `tag.data.shape = "square"`.

### Why It’s Useful in Backend Code

- You keep your API clean and simple.
- You decouple the internal data structure from the public interface.
- You can change how the data is stored later without touching external code.

## Example 2: Validating Configuration

In this second example, let’s say you’re managing application settings like port numbers or environment variables. You want to validate values when they’re set, but keep your class definitions neat.

```python
class PortValidator:
    def __get__(self, instance, owner):
        return instance.__dict__.get('_port')

    def __set__(self, instance, value):
        if not (1024 <= value <= 65535):
            raise ValueError("Port must be between 1024 and 65535")
        instance.__dict__['_port'] = value

    def __delete__(self, instance):
        raise AttributeError("Port can't be deleted")

class AppConfig:
    port = PortValidator()

    def __init__(self, port):
        self.port = port

# Usage
config = AppConfig(8080)
print(config.port)  # 8080
config.port = 70000  # Raises ValueError
```

### So, What’s Happening Again

- `__set__` checks that the port is within a valid range.
- `__get__` fetches the value from instance’s `__dict__`.
- `__delete__` blocks deletion to avoid broken config states.

### And Why It’s Useful in Backend Code

- You enforce data validation at the attribute level.
- You avoid repeating validation logic in multiple places.
- You define consistent rules for critical settings.

## Final Thoughts

The descriptor protocol is a hidden gem in Python that can supercharge your backend design patterns. By hooking into `__get__`, `__set__`, and `__delete__`, you can build smart attributes that:

- Forward access to other layers like data models
- Automatically validate and control data
- Clean up repetitive property definitions

Understanding descriptors might take a little practice, but once you get used to them, they become a very natural and Pythonic way to organize your code.