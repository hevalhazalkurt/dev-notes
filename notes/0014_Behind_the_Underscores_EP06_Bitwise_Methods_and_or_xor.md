# Behind the Underscores EP06: Bitwise Methods (`__and__`, `__or__`, `__xor__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep06-bitwise-methods-__and__-__or__-__xor__/).

<br>

You’ve probably used operators like `&`, `|`, `^`, or `~` in Python before. They're called bitwise operators because they work at the level of binary digits, or bits. So instead of working with whole numbers or booleans, they deal with the 0s and 1s that make up numbers underneath. But have you ever wondered how Python knows what to do when you write `a & b`? Or how you can make your own class respond to those operators?

That’s where bitwise special methods come in, things like `__and__`, `__or__`, `__xor__`, etc. They let you control how bitwise operators work on custom objects, just like how `__add__` lets you control `+`.

Let’s break it all down in simple terms, with code and practical ideas you can actually use.

## What Are Bitwise Special Methods?

Bitwise special methods are part of Python's “dunder methods”. These are the methods you can define in a class to control how operators work.

Here’s a quick list of the main ones related to bitwise operations:

| **Operator** | **Special Method** | **Example Syntax** |
| --- | --- | --- |
| `&` (AND) | `__and__(self, other)` | `a & b` |
| `|`  | `__or__(self, other)`  | `a | b` |
| `^` (XOR) | `__xor__(self, other)` | `a ^ b` |
| `~` (NOT / invert) | `__invert__(self)` | `~a` |
| `<<` (left shift) | `__lshift__(self, other)` | `a << 1` |
| `>>` (right shift) | `__rshift__(self, other)` | `a >> 1` |

Also, Python provides reflected versions for operations where the left operand doesn’t handle the logic:

| Method | Used when |
| --- | --- |
| `__rand__` | If `other.__and__(self)` is used |
| `__ror__` | If `other.__or__(self)` is used |
| `__rxor__` | Same idea for XOR |

## Why Would You Use These?

You might wonder why would I ever override these? Here are a few real-world cases:

- Creating a permissions system like admin, read, write access, using flags.
- Designing custom containers that combine or compare data using binary logic.
- Building a configuration manager where features are toggled on/off with bit flags.
- Making your code look cleaner and more Pythonic by using operators instead of long method calls.

## Example 1: Custom Permissions Class

Let’s build a simple class to manage user permissions using bit flags.

```python
class Permissions:
    def __init__(self, value=0):
        self.value = value  # store as an integer (bit field)

    def __and__(self, other):
        return Permissions(self.value & other.value)

    def __or__(self, other):
        return Permissions(self.value | other.value)

    def __xor__(self, other):
        return Permissions(self.value ^ other.value)

    def __invert__(self):
        return Permissions(~self.value)

    def __str__(self):
        return f"Permissions({bin(self.value)})"

READ = Permissions(0b001)
WRITE = Permissions(0b010)
EXECUTE = Permissions(0b100)

user_perm = READ | WRITE
print(user_perm)  # Permissions(0b11)

admin_perm = user_perm | EXECUTE
print(admin_perm)  # Permissions(0b111)

read_only = admin_perm & READ
print(read_only)  # Permissions(0b1)

no_exec = admin_perm ^ EXECUTE
print(no_exec)  # Permissions(0b11)
```

We’re using bitwise operations to combine permissions (OR), check for overlapping permissions (AND), or toggle one (XOR). This is super common in systems like file permissions, user roles, or feature flags.

## Example 2: Using `__rand__` and Fallback Logic

Sometimes, the left-hand object doesn’t know how to handle the operation, especially if it’s a built-in type like `int`. In that case, Python tries the right-hand operand’s reflected method.

```python
class Flag:
    def __rand__(self, other):
        return f"Reflected AND with {other}"

print(5 & Flag())  # Calls Flag().__rand__(5)
```

Here, Python first tries `5.__and__(Flag())`, which returns `NotImplemented`, so it then tries `Flag().__rand__(5)`. That’s why this fallback system exists.

## Bitwise Shifting: `__lshift__` and `__rshift__`

Sometimes you want to move bits left or right, like multiplying/dividing by powers of 2.

```python
class BitShifter:
    def __init__(self, value):
        self.value = value

    def __lshift__(self, other):
        return BitShifter(self.value << other)

    def __rshift__(self, other):
        return BitShifter(self.value >> other)

    def __str__(self):
        return bin(self.value)

shifter = BitShifter(0b0010)
print(shifter << 1)  # 0b100
print(shifter >> 1)  # 0b1
```

Great for things like image processing, encoding/decoding, or low-level optimization.

## Important Things to Watch Out For

### 1. **Operands must be the same type or compatible**

If your `__and__` method gets something unexpected like an `int`, handle it carefully or return `NotImplemented`.

```python
def __and__(self, other):
    if not isinstance(other, Permissions):
        return NotImplemented
    return Permissions(self.value & other.value)
```

### 2. **Don’t confuse with logical operators**

- `and`, `or`, `not` → used for boolean logic
- `&`, `|`, `~` → used for binary/bitwise operations

They’re completely different things!

### 3. **Bitwise NOT `~` uses two’s complement**

That means `~0b0001` is not `0b1110`, but actually `-0b0010`. It flips all bits, including the sign bit.

## Summary

Bitwise special methods like `__and__`, `__or__`, and `__xor__` give you low-level control over how your custom classes behave with binary operators.

They're great for:

- Permission and access systems
- Efficient storage of toggled states
- Cleaner and more expressive class interfaces
- Working with hardware, image processing, and performance-critical apps

By implementing these dunder methods, your objects can talk to Python’s operators directly and that gives you both power and elegance.