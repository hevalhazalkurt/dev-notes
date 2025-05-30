# Combining Abstract Classes with Factory and Strategy Patterns in Python

Check this post on my blog [here](https://hevalhazalkurt.com/blog/combining-abstract-classes-with-factory-and-strategy-patterns-in-python/).

<br>

When building complex systems, especially in backend development, maintaining flexibility and scalability is key. One powerful way to achieve this is by combining abstract base classes (ABCs) with design patterns like Factory and Strategy. This blog will walk you through this combo, starting with the basics and moving into real-life backend use cases.

## Part 1: The Basics → What Are Abstract Classes?

In Python, abstract classes live in the `abc` module. They're used to define interfaces or templates for other classes to follow. Why do we care? Because abstract classes let you define a common structure and force subclasses to implement specific methods.

Let’s take a look this basic example: 

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def process(self, amount):
        pass
```

Here, `PaymentProcessor` is saying: “Any class that inherits me must implement a `process()` method.” If a subclass forgets to define `process()`, Python will raise an error.

## Part 2: The Factory Pattern + Abstract Classes

The Factory Pattern is about delegating the instantiation of objects to a factory method or class. It works beautifully with ABCs. Let’s say you’re building a SaaS platform that supports multiple payment providers: Stripe, PayPal, and Bank Transfer.

### Step 1: Define the Abstract Base Class

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def process(self, amount):
        pass
```

### Step 2: Create Concrete Implementations

```python
class StripeProcessor(PaymentProcessor):
    def process(self, amount):
        print(f"Processing ${amount} through Stripe.")

class PayPalProcessor(PaymentProcessor):
    def process(self, amount):
        print(f"Processing ${amount} through PayPal.")
```

### Step 3: Create a Factory

```python
def get_payment_processor(provider: str) -> PaymentProcessor:
    if provider == 'stripe':
        return StripeProcessor()
    elif provider == 'paypal':
        return PayPalProcessor()
    else:
        raise ValueError("Unsupported payment provider.")
```

### Usage

```python
processor = get_payment_processor('stripe')
processor.process(100)
```

This setup:

- Decouples your code from concrete classes.
- Makes it easy to plug in new processors later.

## Part 3: Strategy Pattern + Abstract Classes

The Strategy Pattern is about choosing behavior at runtime. Instead of hardcoding logic, you inject it. Say you're building a discount system for an e-commerce backend. Different users have different discount rules.

### Define the Strategy Interface

```python
from abc import ABC, abstractmethod

class DiscountStrategy(ABC):
    @abstractmethod
    def apply_discount(self, amount):
        pass
```

### Implement Strategies

```python
class NoDiscount(DiscountStrategy):
    def apply_discount(self, amount):
        return amount

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percent):
        self.percent = percent

    def apply_discount(self, amount):
        return amount * (1 - self.percent / 100)
```

### Use Strategy in a Context Class

```python
class Checkout:
    def __init__(self, discount_strategy: DiscountStrategy):
        self.discount_strategy = discount_strategy

    def finalize(self, amount):
        return self.discount_strategy.apply_discount(amount)
```

### Usage

```python
checkout = Checkout(PercentageDiscount(10))
print(checkout.finalize(200))  # Output: 180.0
```

Now your business logic is clean, testable, and open to extension.

## A Backend Example: Notification System

Let’s combine both patterns for a notification system in a backend platform that supports Email, SMS, and Slack alerts.

### Step 1: Abstract Base Class for Notification

```python
from abc import ABC, abstractmethod

class Notifier(ABC):
    @abstractmethod
    def send(self, message: str):
        pass
```

### Step 2: Implement Concrete Notifiers

```python
class EmailNotifier(Notifier):
    def send(self, message: str):
        print(f"Sending email: {message}")

class SMSNotifier(Notifier):
    def send(self, message: str):
        print(f"Sending SMS: {message}")

class SlackNotifier(Notifier):
    def send(self, message: str):
        print(f"Sending Slack message: {message}")
```

### Step 3: Factory for Creating Notifiers

```python
def get_notifier(channel: str) -> Notifier:
    if channel == 'email':
        return EmailNotifier()
    elif channel == 'sms':
        return SMSNotifier()
    elif channel == 'slack':
        return SlackNotifier()
    else:
        raise ValueError("Unknown channel.")
```

### Step 4: Strategy for Choosing Notification Time

```python
class NotificationStrategy(ABC):
    @abstractmethod
    def should_notify(self, user_settings: dict) -> bool:
        pass

class NotifyAlways(NotificationStrategy):
    def should_notify(self, user_settings):
        return True

class NotifyBusinessHours(NotificationStrategy):
    def should_notify(self, user_settings):
        from datetime import datetime
        hour = datetime.now().hour
        return 9 <= hour < 17
```

### Step 5: Combine Everything

```python
class NotificationService:
    def __init__(self, notifier: Notifier, strategy: NotificationStrategy):
        self.notifier = notifier
        self.strategy = strategy

    def notify(self, user_settings, message):
        if self.strategy.should_notify(user_settings):
            self.notifier.send(message)
```

### Usage

```python
notifier = get_notifier('email')
strategy = NotifyBusinessHours()
service = NotificationService(notifier, strategy)
service.notify({}, "System maintenance scheduled.")
```

This setup allows you to:

- Easily switch between notifier types.
- Choose different rules for when to notify.
- Unit test each component independently.

## When to Use This Pattern Combo?

Use abstract classes with Factory + Strategy when:

- You have multiple implementations of a concept.
- You want to avoid tight coupling.
- You plan to grow the system like new providers, new rules.
- You care about clean architecture and testability.

## Final Thoughts

Using abstract classes in Python with Factory and Strategy patterns:

- Makes your code flexible and extensible.
- Keeps your business logic clean and decoupled.
- Fits naturally into real-world backend development.

When your system grows, these patterns help bring order to the chaos.