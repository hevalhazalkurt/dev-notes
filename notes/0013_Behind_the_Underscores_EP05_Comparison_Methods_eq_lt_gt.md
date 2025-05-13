# Behind the Underscores EP05: Comparison Methods (`__eq__`, `__lt__`, `__gt__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep05-comparison-methods-__eq__-__lt__-__gt__/).

<br>

In Python, objects aren’t just data containers, they can define how they should be compared with each other. That’s what comparison methods are all about. They allow you to control the meaning of expressions like `a == b`, `a < b`, or `a >= b`.

This post will walk you through these special methods, explain when to use them, how to use them correctly, and what pitfalls to avoid.

## What Are Comparison Methods?

Python uses special (or “magic”) methods to handle built-in operations. For comparisons, these methods start and end with double underscores (`__`), and include:

| **Method** | **Operator it supports** |
| --- | --- |
| `__eq__(self, other)` | `==` |
| `__ne__(self, other)` | `!=` |
| `__lt__(self, other)` | `<` |
| `__le__(self, other)` | `<=` |
| `__gt__(self, other)` | `>` |
| `__ge__(self, other)` | `>=` |

These methods let you define what it means for your own class instances to be “equal”, “less than”, “greater than”, etc.

## Why Would You Need to Override Them?

Here are some use cases where you’d need to define comparison behavior:

### 1. **Sorting Objects**

You want to sort a list of `Student` objects by GPA. But by default, Python has no idea how to compare them. You must define comparison logic using methods like `__lt__`.

### 2. **Deduplication in Sets or Dictionaries**

If you're putting `User` objects into a set or using them as dictionary keys, Python uses `__eq__` and `__hash__` to check for uniqueness.

### 3. **Building Domain-Specific Logic**

Comparing `Version("1.2.3") < Version("2.0.0")` in a software updater? You’ll need to define how versions compare.

## How to Implement Comparison Methods

Let’s look at practical examples for each.

### Example 1: Equality with `__eq__`

```python
class Book:
    def __init__(self, title, author):
        self.title = title
        self.author = author

    def __eq__(self, other):
        if not isinstance(other, Book):
            return NotImplemented
        return self.title == other.title and self.author == other.author
```

This allows you to write:

```python
Book("1984", "Orwell") == Book("1984", "Orwell")  # True
```

**Why return `NotImplemented`?**

Because if `other` isn’t a `Book`, Python should try the reverse comparison (`other.__eq__(self)`) or raise a `TypeError`.

### Ordering with `__lt__` and `@total_ordering`

You don’t have to implement *all* six comparison methods. Python’s `functools.total_ordering` helps you generate the rest if you just provide `__eq__` and one of (`__lt__`, `__gt__`, etc.)

```python
from functools import total_ordering

@total_ordering
class Student:
    def __init__(self, name, gpa):
        self.name = name
        self.gpa = gpa

    def __eq__(self, other):
        return self.gpa == other.gpa

    def __lt__(self, other):
        return self.gpa < other.gpa
```

Now you can do:

```python
alice = Student("Alice", 3.5)
bob = Student("Bob", 3.7)

print(alice < bob)   # True
print(alice >= bob)  # False
```

## Common Pitfalls and Dangers

### 1. **Inconsistent Logic**

If you implement `__eq__` but forget to implement `__hash__`, your objects won’t behave correctly in `set` or as dictionary keys.

```python
class Broken:
    def __eq__(self, other):
        return True
```

```python
s = {Broken(), Broken()}
print(len(s))  # Still 2, because they're unhashable
```

**Rule of thumb**: If your object is immutable and implements `__eq__`, also implement `__hash__`.

### **Violating Transitivity**

Be careful that your comparison logic doesn’t lead to nonsense like:

```python
a < b == c < a  # This should NEVER be True
```

Always make sure your logic is transitive, symmetric, and reflexive, where appropriate.

### **Forgetting Type Checks**

Don't assume the `other` object is the same type:

```python
def __lt__(self, other):
    return self.value < other.value  # Could crash if other is a string or None!
```

Instead:

```python
if not isinstance(other, MyClass):
    return NotImplemented
```

## Real-World Example: Sorting by Multiple Fields

Let’s say you’re building a movie database. You want to sort movies first by rating, then by release year.

```python
@total_ordering
class Movie:
    def __init__(self, title, rating, year):
        self.title = title
        self.rating = rating
        self.year = year

    def __eq__(self, other):
        return (self.rating, self.year) == (other.rating, other.year)

    def __lt__(self, other):
        return (self.rating, self.year) > (other.rating, other.year)  # reverse logic for higher rating
```

```python
movies = [
    Movie("A", 8.5, 2020),
    Movie("B", 8.5, 2019),
    Movie("C", 7.0, 2021)
]

sorted_movies = sorted(movies)
print([m.title for m in sorted_movies])  # ['A', 'B', 'C']
```

Here we’re comparing tuples of attributes for concise and readable logic.

## `__eq__` and `__hash__` : The Dynamic Duo

If you want your objects to work in sets and as dictionary keys, you must implement both `__eq__` and `__hash__` in a consistent way.

```python
class User:
    def __init__(self, username):
        self.username = username

    def __eq__(self, other):
        return self.username == other.username

    def __hash__(self):
        return hash(self.username)
```

```python
u1 = User("alice")
u2 = User("alice")

print(u1 == u2)  # True
print({u1, u2})  # Just one object in the set
```

## Testing Comparison Methods

- Use `==`, `<`, `>`, `<=`, etc. directly in unit tests.
- For sorting, test with `sorted()`.
- Test edge cases: comparing with `None`, different types, self-comparison.

## Summary

- Python lets you define your own comparison logic using `__eq__`, `__lt__`, etc.
- Implement `__eq__` and `__hash__` for equality + hashing (e.g., in sets/dicts).
- Use `@total_ordering` to reduce boilerplate when defining full ordering.
- Always check types inside comparison methods and return `NotImplemented` if necessary.
- Be mindful of logic consistency and avoid subtle bugs in sorting or deduplication.