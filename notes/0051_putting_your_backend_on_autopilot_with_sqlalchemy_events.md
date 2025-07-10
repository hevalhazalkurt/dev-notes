# Putting Your Backend on Autopilot with SQLAlchemy Events

Check this post on my blog [here](https://hevalhazalkurt.com/blog/putting-your-backend-on-autopilot-with-sqlalchemy-events/).

<br>


There's a certain point in every backend project where the pristine beauty of your initial code starts to get... complicated. You're crafting a masterpiece, maybe a social platform for artisanal cheese enthusiasts or a booking system for competitive napping. You're using the glorious SQLAlchemy as your ORM, and life is good. Your models are defined, your relationships are solid, and your queries are... well, they're queries and they’re just OK.

But then, the business logic starts to creep in.

- “When a new user signs up, we need to hash their password”
- “Oh, and after they're successfully created, we must send them a welcome email”
- “And every time a user's record is updated, we have to update the `updated_at` timestamp”
- “Actually, can we log every single time a user's email address is changed, for audit purposes?”

Suddenly, your beautiful, clean service layer or API endpoint starts to look like a messy kitchen after a food fight. You have password hashing logic here, email sending logic there, and timestamp updates sprinkled all over the place. It works, but it feels... fragile. Cluttered.

Of course, the challenges you’d face in a real-world project would be much more complex than these. But for the sake of clarity and keeping this blog post approachable, we need to look at things from a simpler, more foundational perspective. So think of these examples as a starting point. 

As a backend engineer, the problems you need to solve or the services you need to build may vary, but one thing’s for sure: sooner or later, you'll need a system that can track changes in the database and react accordingly. That’s where ORM events come into play. In this post, we’ll take a closer look at the event system provided by SQLAlchemy and what it allows us to do.

# The scenario → Welcome to “CodeCrafters”

Throughout this post, we'll be building the user management system for a hypothetical platform called “CodeCrafters”, a place for developers to collaborate and share projects. We'll focus on our main character: the `User` model.

Let's start with our basic setup. We're using SQLAlchemy with a PostgreSQL database.

```python
# models.py
import datetime
from sqlalchemy import (
    create_engine,
    Column,
    Integer,
    String,
    DateTime,
)
from sqlalchemy.orm import sessionmaker, declarative_base
from sqlalchemy import event

# Standard SQLAlchemy setup
DATABASE_URL = "postgresql://user:password@localhost/codecrafters_db"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Our main character: The User model
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True, nullable=False)
    email = Column(String, unique=True, index=True, nullable=False)
    password_hash = Column(String, nullable=False)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.datetime.utcnow)

    def __repr__(self):
        return f"<User(id={self.id}, username='{self.username}')>"

# Some placeholder functions we'll use later
def hash_password(password: str) -> str:
    print(f"--> HASHING password for user...")
    # In a real app, use something like bcrypt
    return f"hashed_{password}"

def send_welcome_email(user_email: str):
    print(f"--> Sending welcome email to {user_email}...")
    # Imagine a real email sending logic here
    print("--> Email sent!")
```

Simple enough, right? Now, let's give this model some superpowers.

# The two flavors of events → Mapper vs. Session

Before we dive headfirst into code, it's crucial to understand that SQLAlchemy gives us two main arenas to play in: Mapper Events and Session Events. Choosing the right one for the job is the key to keeping your code clean and your logic sound.

### **Mapper Events**

Mapper Events are the personal assistants for a single model. They are listeners that you attach directly to a specific mapped class, like our `User` model. They are obsessed with the lifecycle of only that type of object.

**When to use them:** The rule of thumb here is simple: if the logic you're writing only makes sense in the context of a `User` or a `Product`, or a `BlogPost`, a Mapper Event is your best bet.

- **Hashing a user's password?** That's a `User` problem. It makes no sense for a `Project` model.
- **Sending a welcome email to a new user?** Again, this is a `User`-specific action.
- **Validating that a username follows a specific format?** That's a `User`'s concern.

These events are for logic that is tightly coupled to the identity and responsibilities of a single model.

### **Session Events**

If Mapper Events are the specialists, Session Events are the general managers. They don't care about any single model; they listen to the `Session` object itself. They have a bird's-eye view of the entire unit of work; all the new objects, all the modified ones, and all the deleted ones, regardless of their class.

**When to use them:** This is your go-to when you have logic that should apply globally or to a wide, heterogeneous range of models.

- **Automatically updating an `updated_at` timestamp?** This is a classic. Your `User`, `Project`, and `Comment` models might all have this field.
- **Logging all database changes to a central audit trail?** A Session Event can inspect every object in `session.dirty` or `session.new` and create a generic log entry.
- **Invalidating an external cache after a transaction successfully commits?** The `after_commit` Session Event is perfect, as it ensures the database write was successful before you take an external action.

So, as we explore the examples, ask yourself this simple question: “Is this logic inherently part of being a `User`, or is it a general rule for my entire database session?”

- Specific to the model? -> Mapper Event.
- General, cross-model rule? -> Session Event.

# Let’s start building our app

## Part 1 → Mapper Events

Mapper Events are listeners that are tied to a specific mapped class like our `User` model. They fire when instances of that class go through different states in their lifecycle within a `Session`. Think of them as your model's personal bodyguards, checking things before and after major events.

### **The `before_insert` event: The gatekeeper**

Our first requirement is hashing the user's password before saving it to the database. We should never store plain-text passwords. This is a perfect job for a `before_insert` event. We can attach a listener to our `User` class that will intercept any new `User` object right before the `INSERT` statement is sent to Postgres.

```python
@event.listens_for(User, 'before_insert')
def hash_user_password(mapper, connection, target: User):
    """Listen for the 'before_insert' event on the User model.
    
    The 'target' is the actual User instance being processed.
    """
    print(f"EVENT: 'before_insert' on {target}")
    # We'll get a 'password' attribute from the user object, hash it,
    # and store it in 'password_hash'
    # In a real app, you'd handle the temporary password attribute more cleanly.
    if hasattr(target, 'password'):
        target.password_hash = hash_password(target.password)
        del target.password # Don't keep the plain-text password in memory
```

Let's see this in action.

```python
# main.py
from models import User, SessionLocal

db = SessionLocal()

print("Creating a new user...")
# We add a temporary 'password' attribute that our listener will see
new_user = User(username="ada_lovelace", email="ada@example.com")
new_user.password = "supersecret123" 

db.add(new_user)
db.commit()

print(f"User created: {new_user}")
print(f"Password in DB: {new_user.password_hash}")
```

**Output:**

```bash
Creating a new user...
EVENT: 'before_insert' on <User(id=None, username='ada_lovelace')>
--> HASHING password for user...
User created: <User(id=1, username='ada_lovelace')>
Password in DB: hashed_supersecret123
```

Look at that! Our application code didn't have to know anything about hashing. It just created a `User` object. The event listener, attached directly to the `User` model's lifecycle, took care of the security logic automatically. Clean, decoupled, and reusable.

So, why is this `before_insert` listener the non-negotiable choice for our password hashing? Think of this event as the final quality check on the assembly line. It fires after you've called `db.add(new_user)` but right before `db.commit()` actually translates that object into an `INSERT` statement to be sent to Postgres. At this moment, our `new_user` object is fully formed in Python memory, but it doesn't have an id yet and hasn't touched the database.

This timing is golden. It's our last, best chance to intercept the temporary plaintext password attribute we created, perform the one-way hashing operation, and place the result safely into the `password_hash` field. If we tried to do this after the insert (`after_insert`), it would be too late! We would have already attempted to commit an object without a valid hash, which would likely violate a `NOT NULL` constraint, or even worse, risk leaking the plaintext password into a log or error message. `before_insert` is not just for security; it's perfect for any kind of pre-save data preparation, like generating a URL-friendly slug from a blog post title or forcing an email address to lowercase before it's saved. It ensures our user object is database-ready and properly formatted before it ever leaves the application.

### **The `after_insert` event: The welcoming committee**

Next requirement is sending a welcome email after the user is created. Why `after_insert` and not `before_insert`? Because we want this to happen only if the database transaction is successful. If the `INSERT` fails (e.g., due to a duplicate email), we don't want to send an email. Also, after the insert, our `new_user` object will be populated with its id from the database, which can be useful for logging or other operations.

```python
@event.listens_for(User, 'after_insert')
def send_user_welcome_email(mapper, connection, target: User):
    """Listen for the 'after_insert' event."""
    print(f"EVENT: 'after_insert' on {target}")
    # The user is now safely in the database. Time to send the email!
    send_welcome_email(target.email)
```

When we run our `main.py` script again, the output will now include the email sending step, which fires after the `before_insert` hashing logic.

### **The `before_update` event: The timekeeper**

What about our `updated_at` timestamp? We want it to be automatically updated whenever we change a user's record. A `before_update` event is perfect for this.

```python
@event.listens_for(User, 'before_update')
def update_timestamp(mapper, connection, target: User):
    """Listen for the 'before_update' event."""
    print(f"EVENT: 'before_update' on {target}")
    # Update the updated_at field to the current time
    target.updated_at = datetime.datetime.utcnow()
```

Now, let's try updating a user.

```python
# main.py continued...
user_to_update = db.query(User).filter_by(username="ada_lovelace").one()
print(f"\nUpdating user {user_to_update.username}...")
print(f"Timestamp before update: {user_to_update.updated_at}")

user_to_update.email = "ada.lovelace@newdomain.com"
db.commit()

print(f"Timestamp after update:  {user_to_update.updated_at}")
```

**Output:**

```bash
Updating user ada_lovelace...
Timestamp before update: 2023-10-27 10:30:00.123456
EVENT: 'before_update' on <User(id=1, username='ada_lovelace')>
Timestamp after update:  2023-10-27 10:30:05.987654
```

It works like a charm! The timestamp is updated automatically before the `UPDATE` statement is executed, ensuring our record is always fresh.

## **Part 2: The `propagate` flag, a family affair**

What happens if we have inheritance? Let's say CodeCrafters introduces a `PremiumUser` who inherits from `User`.

```python
class PremiumUser(User):
    __tablename__ = 'premium_users'
    id = Column(Integer, primary_key=True)
    # ... other premium features
    __mapper_args__ = {
        'polymorphic_identity': 'premium_user',
    }
```

By default, all events attached to `User` will also apply to `PremiumUser`. This is because the `propagate=True` flag is set by default on `@event.listens_for`. The password hashing, for example, is something we definitely want for all types of users. 

But what if we wanted to run a specific event only for the base `User` class and not its children? We can set `propagate=False`.

```python
@event.listens_for(User, 'after_insert', propagate=True) # Default, good for password hashing
def hash_all_users_password(...): ...

@event.listens_for(User, 'after_insert', propagate=False) # Only for base User
def send_standard_welcome_email(mapper, connection, target):
    """This will NOT run for PremiumUser instances."""
    if type(target) is User:
        print("--> Sending STANDARD welcome email.")
```

This gives you fine-grained control over how events behave in an inheritance hierarchy, a powerful feature for complex domain models.

## Part 3: Attribute events

Sometimes, listening to the entire object being updated is overkill. What if we only care when a specific attribute changes? This is the use case for Attribute Events.

Our final requirement was log every time a user's email address is changed.

```python
@event.listens_for(User.email, 'set')
def log_email_change(target: User, value, oldvalue, initiator):
    """Listen for the 'set' event on the User.email attribute."""
    # The 'oldvalue' can be a special marker if the attribute wasn't loaded before
    print(f"ATTRIBUTE EVENT: User {target.username}'s email is being changed!")
    print(f"  Old email: {oldvalue}")
    print(f"  New email: {value}")
    # Here you could write to an audit log table
```

The `set` event fires whenever a value is assigned to that attribute. The listener function receives the target instance, the value being set, the `oldvalue`, and the initiator of the event.
Running our update script from before now gives us even more detailed output:

```bash
Updating user ada_lovelace...
Timestamp before update: 2023-10-27 10:30:00.123456
ATTRIBUTE EVENT: User ada_lovelace's email is being changed!
  Old email: ada@example.com
  New email: ada.lovelace@newdomain.com
EVENT: 'before_update' on <User(id=1, username='ada_lovelace')>
Timestamp after update:  2023-10-27 10:30:05.987654
```

Notice the order: the attribute event fires the moment we do `user_to_update.email = "..."`, even before the `before_update` event, which fires just before the `commit()`. This precision is incredibly useful for validation, auditing, or creating derived values.

## Part 4: Session events, the big picture

So far, we've been attaching listeners to specific models. But what if you have a piece of logic that should apply to many different models? For example, almost every table in our database might have `created_at` and `updated_at` columns. Attaching a `before_update` listener to every single model would violate the DRY (Don't Repeat Yourself) principle.

This is where **Session Events** come in. They listen to the `Session` object itself, not a specific model. The most powerful of these is `before_flush`.

The flush process is when SQLAlchemy figures out all the `INSERT`, `UPDATE`, and `DELETE` statements it needs to send to the database based on the changes you've made to your objects in the session. before_flush is your last chance to inspect and modify these objects before they are translated into SQL.

Let's refactor our timestamp logic into a single, elegant session event.

```python
class TimestampMixin:
    created_at = Column(DateTime, default=datetime.datetime.utcnow, nullable=False)
    updated_at = Column(DateTime, default=datetime.datetime.utcnow, nullable=False)

# Update our User model to use it
class User(Base, TimestampMixin):
    # ... (remove the explicit created_at/updated_at columns)
    ...

# Now, the session event. You can define this anywhere your session is configured.
@event.listens_for(SessionLocal, 'before_flush')
def universal_timestamp_listener(session, flush_context, instances):
    """Listen for the 'before_flush' event on any session."""
    print("SESSION EVENT: 'before_flush'")
    # Iterate over all new and modified objects in the session
    for instance in session.new.union(session.dirty):
        if isinstance(instance, TimestampMixin):
            # If the object has our mixin, update its timestamp
            instance.updated_at = datetime.datetime.utcnow()
            # If it's a new object, also set the created_at timestamp
            if instance in session.new:
                instance.created_at = datetime.datetime.utcnow()
```

Now, we can remove the `before_update` and `before_insert` listeners from the `User` model that were handling timestamps. This one `before_flush` event will automatically manage `created_at` and `updated_at` for any model that uses our `TimestampMixin`. This is massively scalable and keeps your models incredibly clean.

Other useful session events include:

- `after_commit`: Fire tasks that should only happen after a transaction is successfully written to the database. For example, clearing an external cache, enqueuing a background job etc.
- `after_rollback`: Perform cleanup if a transaction fails.

## So, why you should care

SQLAlchemy Events are more than just a neat trick. They are a fundamental pattern for building robust, maintainable, and intelligent data layers.

- **Decoupling:** Your application/service logic doesn't need to know about implementation details like password hashing or sending emails. It just manages objects.
- **Centralization:** All the logic related to a model's lifecycle lives with the model, not scattered across your codebase.
- **Automation:** You can automate repetitive tasks like updating timestamps or creating audit logs with incredible elegance.
- **Power & Precision:** From broad session-wide events to hyper-specific attribute-level listeners, you have the exact tool for the job.

So, the next time you find yourself writing an if `new_object`, block in your API endpoint, take a step back and ask: “Could an ORM event handle this for me?” The answer will often be a resounding “Yes”, and your future self will thank you for the cleaner, smarter code.