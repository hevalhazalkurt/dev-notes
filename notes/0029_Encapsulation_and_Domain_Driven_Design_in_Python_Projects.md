# Encapsulation and Domain-Driven Design in Python Projects

Check this post on my blog [here](https://hevalhazalkurt.com/blog/encapsulation-and-domain-driven-design-in-python-projects/).

<br>

When you hear the word “encapsulation”, you might think of private variables or hiding attributes in classes. And that’s partly true. But in real-world Python projects, especially large backend systems, encapsulation plays a much deeper role. It helps us design systems that are easier to change, test, and understand. And when you combine it with Domain-Driven Design (DDD) principles, you can build robust, maintainable applications that actually model the complexity of your business logic.

This blog post will walk you through the concept of encapsulation in Python, show how it applies in Domain-Driven Design, and give you practical, examples from backend systems.

## What Is Encapsulation, Really?

Encapsulation is about hiding the internal state and requiring all interactions to go through well-defined interfaces. In Python, we can't truly make things private like in Java or C++. But we can mark something as “private by convention” using a leading underscore (`_`) or make it strongly suggested private with double underscore name mangling (`__`).

Let’s take a look at a basic example →

```python
class User:
    def __init__(self, username, password):
        self.username = username
        self.__password = password  # private

    def check_password(self, input_password):
        return self.__password == input_password
```

In this example, `__password` is intended to be private. You don't want other parts of the system to modify or read it directly. Instead, you provide a method `check_password()` that safely compares a given input. This keeps your internal logic safe from accidental tampering and provides a clear interface for how other code should interact with the object.

But what about a real-world app?

## Encapsulation in Backend Services

In backend projects, we encapsulate not just data, but business rules, invariants, and logic flows. Here’s a more realistic example: managing a user account.

```python
class User:
    def __init__(self, username):
        self.username = username
        self.__is_active = False

    def activate(self):
        self.__is_active = True

    def deactivate(self):
        self.__is_active = False

    def is_active(self):
        return self.__is_active
```

In this version, instead of letting external code flip the `is_active` flag directly, we enforce all state changes through methods. This gives us the power to later add logic like sending an email when a user is activated or logging the action. The actual boolean flag is protected inside the object, ensuring consistency and centralizing rule enforcement.

## Introducing Domain-Driven Design (DDD)

DDD is a design philosophy that focuses on modeling the real-world domain in code. It encourages dividing your codebase into:

- **Entities**: objects with a unique identity and lifecycle like User, Order
- **Value Objects**: immutable objects with no identity like Money, DateRange
- **Aggregates**: clusters of objects treated as a unit
- **Repositories**: interfaces for accessing aggregates
- **Services**: operations that don't naturally belong to a specific entity

Encapsulation in DDD means: hide internal state and enforce all rules through methods on the domain object.

## Example: An Order System with Encapsulation

Let’s model an order that has line items, a status, and a business rule: you can only cancel if it's not shipped.

```python
class Order:
    def __init__(self, order_id):
        self.order_id = order_id
        self.__items = []
        self.__status = "draft"

    def add_item(self, item):
        if self.__status != "draft":
            raise Exception("Can't add items after order is confirmed")
        self.__items.append(item)

    def confirm(self):
        if not self.__items:
            raise Exception("Can't confirm empty order")
        self.__status = "confirmed"

    def cancel(self):
        if self.__status == "shipped":
            raise Exception("Can't cancel a shipped order")
        self.__status = "canceled"

    def ship(self):
        if self.__status != "confirmed":
            raise Exception("Can't ship unless confirmed")
        self.__status = "shipped"

    def status(self):
        return self.__status
```

Why this is good encapsulation

- Business rules are enforced inside the object.
- No external code can make the order “shipped” without going through the `ship()` method.
- Side effects like logging or emails can be added to the `cancel()` method, and all callers benefit.

## Value Object: Money

Let’s say you want to represent prices safely.

```python
class Money:
    def __init__(self, amount, currency):
        if amount < 0:
            raise ValueError("Amount can't be negative")
        self.amount = amount
        self.currency = currency

    def __add__(self, other):
        if self.currency != other.currency:
            raise ValueError("Currency mismatch")
        return Money(self.amount + other.amount, self.currency)
```

This class is a value object. It has no identity, and it's defined only by its content. You use `Money` instead of raw floats or integers to make sure every piece of price-related logic goes through this safe structure. For instance, you can prevent someone from accidentally mixing EUR and USD values, or from passing a negative price into your system.

## Advanced Pattern: Aggregate Roots and Invariants

In DDD, an aggregate root is the main entity that guards the consistency of related objects. Let’s extend our `Order` to also manage shipments. Instead of letting another class touch shipment state, the `Order` itself enforces the rules.

```python
class Shipment:
    def __init__(self):
        self.__shipped = False

    def mark_shipped(self):
        self.__shipped = True

    def is_shipped(self):
        return self.__shipped

class Order:
    def __init__(self, order_id):
        self.order_id = order_id
        self.__status = "draft"
        self.__shipment = Shipment()

    def ship(self):
        if self.__status != "confirmed":
            raise Exception("Can't ship unless confirmed")
        self.__shipment.mark_shipped()
        self.__status = "shipped"
```

Here, the `Shipment` class is completely hidden inside the `Order` object. External callers can't directly call `mark_shipped()` on the shipment. This means the only way to change shipment state is by calling `Order.ship()`. That way, we ensure the order's status and the shipment's status stay in sync.

## Common Mistakes When Encapsulating

- Making fields public “just to make testing easier”
- Allowing direct database models like SQLAlchemy to leak into business logic
- Writing services that manipulate raw dicts instead of rich objects

Encapsulation isn't just a language feature, it's a discipline.

## Takeaways

- Encapsulation in Python goes beyond hiding variables; it's about protecting your domain logic.
- Domain-Driven Design gives you a structure to group and encapsulate logic meaningfully.
- In real-world backends, encapsulated entities and aggregates make your codebase more testable and maintainable.

If you're working on a system that models real business processes, encapsulation + DDD is your best friend.