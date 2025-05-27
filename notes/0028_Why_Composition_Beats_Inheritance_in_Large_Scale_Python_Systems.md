# Why Composition Beats Inheritance in Large-Scale Python Systems

Check this post on my blog [here](https://hevalhazalkurt.com/blog/why-composition-beats-inheritance-in-large-scale-python-systems/).

<br>


When you start out learning object-oriented programming (OOP) in Python, inheritance feels like the obvious way to reuse code. You build a base class, extend it, and reuse its behavior. But as your software grows, you might start noticing that inheritance introduces tight coupling, rigid structures, and eventually... a mess.

This post is about why composition is often a better choice than inheritance in large-scale Python systems, especially when you're building backend applications. We’ll start from the basics, then go into real-life backend scenarios to show how composition can help you write cleaner, more maintainable code.

## Inheritance vs Composition

**Inheritance** means creating a new class that is a type of another class. It forms an "is-a" relationship. The child class (subclass) automatically gets the properties and methods of the parent (base) class. This is great when the subclass truly is a specialized form of the parent.

```python
class Animal:
    def speak(self):
        return "Some sound"

class Dog(Animal):
    def speak(self):
        return "Woof!"
```

Here, `Dog` is an `Animal`, so it makes sense to inherit. But if you start inheriting just to reuse `speak()`, that's a red flag, you're misusing inheritance.

**Composition**, on the other hand, means building a class using other classes often by passing them in as attributes. It forms a “has-a” relationship. One class delegates behavior to another, which makes the structure more flexible and modular.

```python
class Engine:
    def start(self):
        return "Engine started"

class Car:
    def __init__(self, engine):
        self.engine = engine

    def start(self):
        return self.engine.start()
```

Here, `Car` has an `Engine`, and it uses the engine's behavior without being tightly bound to its implementation. This makes testing, swapping, or extending `Engine` behavior much easier.

To sum it up:

- Use **inheritance** when you have a clear hierarchical relationship.
- Use **composition** when you want flexibility, testability, and better separation of concerns.

## The Problems with Inheritance in Big Codebases

In small systems, inheritance can work fine. But once you scale up, it starts to cause trouble:

### 1. **Tight Coupling**

Child classes depend on the structure of the parent. If you change the base class, you risk breaking subclasses.

### 2. **Inheritance Hierarchies Get Deep and Confusing**

When you have a base class, a child class, a grandchild class... it's hard to track where methods are coming from.

### 3. **Inheritance Isn’t Flexible**

What if you need to reuse the same behavior in two unrelated classes? Inheritance can’t help without breaking the “is-a” rule.

### 4. **Testing Becomes Harder**

When behavior is inherited from many levels up, writing isolated unit tests becomes a pain.

## Composition to the Rescue

With composition, you build classes that contain instances of other classes. This way, your classes are like Lego blocks: reusable, testable, and loosely coupled. Let’s walk through some backend scenarios where composition shines.

## Scenario 1: Injecting Services into API Endpoints

When building real-world APIs, for example with FastAPI or Flask, you often need to integrate services like email delivery, logging, analytics, or third-party APIs. A common mistake is to tightly bind these services into the route or controller logic, or worse, to subclass everything in an attempt to "reuse" behavior. That’s where composition becomes a huge win.

Let's say your API should send a welcome email when a new user signs up. You might be tempted to create a `BaseSignupHandler` with a `send_email` method, and then subclass it. But that approach hardwires the behavior and makes testing or swapping the email logic a pain.

Instead, you can define a dedicated service class:

```python
class EmailService:
    def send(self, to, subject, body):
        # Send email logic here
        print(f"Sending to {to}: {subject}")

class UserSignupHandler:
    def __init__(self, email_service: EmailService):
        self.email_service = email_service

    def signup(self, user_email):
        # Create user...
        self.email_service.send(
            to=user_email,
            subject="Welcome!",
            body="Thanks for signing up."
        )
```

This gives you several advantages:

**Testability**: You can mock or stub the email service without changing `UserSignupHandler`:

```python
class MockEmailService:
    def send(self, to, subject, body):
        print("Mock send")
```

Now your tests can look like:

```python
handler = UserSignupHandler(MockEmailService())
handler.signup("test@example.com")
```

**Flexibility**: Tomorrow you may want to send Slack notifications or SMS instead of email. With composition, you can simply replace `EmailService` with another implementation. No inheritance gymnastics required.

**Decoupling**: `UserSignupHandler` doesn't care how the message is sent. It only knows it can delegate to the service object. This keeps business logic clean and isolated.

In backend systems, you'll commonly inject:

- Database repositories
- Messaging clients like Kafka, RabbitMQ
- External API clients like Stripe, AWS SDKs
- Logger or metrics providers

Each of these is a perfect candidate for composition.

By embracing composition for service injection, you’re not only making your code easier to maintain and test. You're also aligning with modern software engineering practices like dependency injection and separation of concerns.

Say you’re writing a FastAPI app that has to send emails. Instead of subclassing some `EmailSenderBase`, you use composition:

## Scenario 2: Payments with Strategy Pattern via Composition

Let’s say your app supports multiple payment gateways:

```python
class StripePayment:
    def charge(self, amount):
        print(f"Charging ${amount} using Stripe")

class PaypalPayment:
    def charge(self, amount):
        print(f"Charging ${amount} using PayPal")

class PaymentProcessor:
    def __init__(self, strategy):
        self.strategy = strategy

    def pay(self, amount):
        return self.strategy.charge(amount)
```

You can switch strategies at runtime:

```python
processor = PaymentProcessor(StripePayment())
processor.pay(100)  # Uses Stripe

processor.strategy = PaypalPayment()
processor.pay(200)  # Now uses PayPal
```

This level of flexibility is very hard to achieve with inheritance.

## Scenario 3: Repository Pattern with Swappable Storage

Let’s say you want to store user data, but don’t want your business logic to care whether it’s PostgreSQL or Redis.

```python
class PostgresUserRepo:
    def get_user(self, user_id):
        return {"id": user_id, "name": "Postgres User"}

class RedisUserRepo:
    def get_user(self, user_id):
        return {"id": user_id, "name": "Redis Cached User"}

class UserService:
    def __init__(self, user_repo):
        self.user_repo = user_repo

    def load_user(self, user_id):
        return self.user_repo.get_user(user_id)
```

In tests, you might use an in-memory version:

```python
class InMemoryUserRepo:
    def get_user(self, user_id):
        return {"id": user_id, "name": "Test User"}
```

## Advantages of Composition in Production Code

1. **Loose Coupling**
    
    You can change internal parts without touching other parts of the system.
    
2. **Easier Testing**
    
    Just inject mock or fake objects.
    
3. **More Reuse**
    
    Composable behaviors can be shared across unrelated classes.
    
4. **Better Code Organization**
    
    Each class has one responsibility. You don’t need to dig through long inheritance trees.
    
5. **Dynamic Behavior**
    
    You can change or decorate behavior at runtime.
    

## So..

In short, inheritance is great when there is a true "is-a" relationship. But in real-world backend code, that’s rare. What you really need is flexible, maintainable, and testable code. That’s where composition shines.