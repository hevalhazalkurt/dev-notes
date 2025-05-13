# Behind the Underscores EP04: Arithmetic Methods (`__add__`, `__sub__`, `__mul__`)

Check this post on my blog [here](https://hevalhazalkurt.com/blog/behind-the-underscores-ep04-arithmetic-methods-__add__-__sub__-__mul__/).

<br>

Have you ever wished your Python objects could behave like numbers? What if you could add two custom objects with `+`, or even write `my_obj *= 2` and have it make perfect sense?

Well, Python lets you do that and it’s all thanks to arithmetic magic methods like `__add__`, `__sub__`, and `__mul__`.

In this blog, we’ll go from basics to advanced, with examples you can use right away. We'll also talk about the hidden dangers of these features and how to use them properly.

## What Are Arithmetic Magic Methods?

These are special methods in Python that let your custom objects respond to built-in arithmetic operators like `+`, `-`, `*`, `/`, `**`, and so on.

You’ve probably seen methods like `__init__`, `__str__`, or `__repr__`. Well, `__add__` and its siblings are in the same family, they just deal with math.

### Here's a short list:

| Operator | Method | Example |
| --- | --- | --- |
| `+` | `__add__` | `a + b` |
| `-` | `__sub__` | `a - b` |
| `*` | `__mul__` | `a * b` |
| `/` | `__truediv__` | `a / b` |
| `//` | `__floordiv__` | `a // b` |
| `%` | `__mod__` | `a % b` |
| `**` | `__pow__` | `a ** b` |

There are also reflected versions like `__radd__`, and in-place ones like `__iadd__`.

## Why Do We Need Them?

Let’s say you’re writing a class for money, like `Money(10, 'USD')`. It’d be cool if you could add two money objects like this:

```python
usd1 = Money(10, 'USD')
usd2 = Money(5, 'USD')

print(usd1 + usd2)  # Should print: 15 USD
```

Out of the box, Python has no idea what `+` means for your custom class. That’s where `__add__` comes in. By defining it, you teach Python: “Here’s what it means to add two of my objects.” So instead of writing clunky methods like `money.add(other)`, you can use clean, readable expressions like `money1 + money2`.

## So Let’s Build The Money Class

Let’s walk through a real case step-by-step.

### Step 1: Basic `Money` class with `__add__`

```python
class Money:
    def __init__(self, amount, currency):
        self.amount = amount
        self.currency = currency

    def __str__(self):
        return f"{self.amount} {self.currency}"

    def __add__(self, other):
        if self.currency != other.currency:
            raise ValueError("Can't add different currencies!")
        return Money(self.amount + other.amount, self.currency)

m1 = Money(10, 'USD')
m2 = Money(5, 'USD')
print(m1 + m2)  # 15 USD
```

1. We create two `Money` objects: `m1` has 10 USD and `m2` has 5 USD.
2. When you use `m1 + m2`, Python internally calls `m1.__add__(m2)`.
3. Inside `__add__`, we:
    - Check that both currencies match. If not, we raise an error.
    - Add the amounts: `10 + 5 = 15`.
    - Create and return a new `Money` object with 15 USD.
4. The `__str__` method controls how the object is displayed when printed, so it shows `"15 USD"`.

This example shows how to safely define the meaning of `+` for a class and avoid invalid operations like mixing currencies.

### Step 2: Adding `__radd__` for `int + Money`

```python
class Money:
    ...
    def __radd__(self, other):
        if isinstance(other, int):
            return Money(self.amount + other, self.currency)
        return NotImplemented

print(5 + m1)  # 15 USD
```

1. Normally, `5 + m1` would fail, because Python tries to call `int.__add__(m1)` but `int` doesn’t know how to handle a `Money` object.
2. Python then tries the reflected version: `m1.__radd__(5)`.
3. Inside `__radd__`:
    - We check if the left-hand operand (`other`) is an `int`.
    - If it is, we add that integer to `self.amount`.
    - We return a new `Money` object with the updated amount.

This lets users add raw numbers to your custom objects without worrying about order (`obj + 5` or `5 + obj` both work).

**What Happens If You Skip `__radd__`?**

Let’s remove `__radd__` from the `Money` class and run:

```python
m1 = Money(10, 'USD')
print(5 + m1)

TypeError: unsupported operand type(s) for +: 'int' and 'Money'
```

### Step 3: Adding Supporting `+=` with `__iadd__`

```python
class Money:
    ...
    def __iadd__(self, other):
        if self.currency != other.currency:
            raise ValueError("Different currencies")
        self.amount += other.amount
        return self

m1 = Money(10, 'USD')
m1 += Money(2, 'USD')
print(m1)  # 12 USD
```

1. The `+=` operator in Python uses `__iadd__`, if defined.
2. Here, `m1 += Money(2, 'USD')` is the same as calling `m1.__iadd__(Money(2, 'USD'))`.
3. Inside `__iadd__`:
    - We check that the currencies match just like in `__add__`.
    - But instead of creating a new object, we modify the existing one (`self.amount += ...`).
    - Finally, we return `self`, as Python expects that from `__iadd__`.

This method allows in-place addition. Useful if your object is mutable (which means it can change over time) and you want to avoid making new objects every time.

## What About `__radd__` and `__iadd__`?

- `__radd__` is called when the left object doesn’t know how to add, so Python tries the right one.
- `__iadd__` handles the `+=` operator.

| **Operator** | **What Python Calls** | **Who Gets Called First** |
| --- | --- | --- |
| `a + b` | `a.__add__(b)` | Left-hand operand (`a`) |
| `b + a` | `b.__radd__(a)` | Right-hand operand (`b`) |
| `a += b` | `a.__iadd__(b)` | Left-hand, possibly mutates itself |

## Common Dangers and Gotchas

### 1. Forgetting `__radd__`

If you don’t define `__radd__`, expressions like `5 + obj` will fail, even if `obj + 5` works.

### 2. Mutability Surprise with `__iadd__`

If your object is mutable, `+=` may change the original. If it’s immutable, `+=` just creates a new one. Be explicit to avoid confusion.

### 3. Returning Wrong Types

Always return the same kind of object, or Python may behave weirdly (like returning `int` from `__add__` when it should be a `Money`).

### 4. Inconsistent Logic

If `a + b` works but `b + a` doesn’t, you’ll confuse your future self (or other devs). Make sure your logic is symmetric when it should be.

## Another Example: A Vector Class

Let’s try a more mathy example, a 2D vector.

```python
class Vector2D:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):
        return Vector2D(self.x + other.x, self.y + other.y)

    def __sub__(self, other):
        return Vector2D(self.x - other.x, self.y - other.y)

    def __mul__(self, scalar):
        return Vector2D(self.x * scalar, self.y * scalar)

    def __rmul__(self, scalar):
        return self.__mul__(scalar)

    def __str__(self):
        return f"({self.x}, {self.y})"

v1 = Vector2D(1, 2)
v2 = Vector2D(3, 4)

print(v1 + v2)     # (4, 6)
print(v1 - v2)     # (-2, -2)
print(v1 * 3)      # (3, 6)
print(2 * v2)      # (6, 8)
```

- `__add__`: adds the x and y coordinates of two vectors.
    - `(1 + 3, 2 + 4)` → `(4, 6)`
- `__sub__`: subtracts the x and y coordinates.
    - `(1 - 3, 2 - 4)` → `(-2, -2)`
- `__mul__`: multiplies both coordinates by a scalar.
    - `(1 * 3, 2 * 3)` → `(3, 6)`
- `__rmul__`: ensures `scalar * vector` works (not just `vector * scalar`) by redirecting to `__mul__`.

This class shows how you can fully overload arithmetic operators to model real-world entities like vectors, physics values, or even color channels.

## Best Practices

- Always return a new instance for `__add__`, `__sub__`, etc. unless you have a strong reason not to.
- Use `NotImplemented` if you can’t handle a type. This lets Python try other methods like `__radd__`.
- Match the behavior of Python’s built-in types, people expect them.
- Document clearly what operators your class supports.

## Conclusion

Arithmetic magic methods make your classes feel natural and powerful, almost like they’re part of the language. Whether you’re building a math library, a financial model, or even a game, these methods let you write clean, expressive code.

But with great power comes great responsibility. Use them wisely, document clearly, and always test your logic from both sides.