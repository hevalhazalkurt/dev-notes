# Designing Reusable and Scalable ORM Models with Declarative Base and Mixins

Check this post on my blog [here](https://hevalhazalkurt.com/blog/designing-reusable-and-scalable-orm-models-with-declarative-base-and-mixins/).

<br>

When you're building a backend with an ORM like SQLAlchemy, your models are the heart of the system. But as your project grows, writing every model from scratch becomes painful, repetitive, and error-prone. That’s where reusable models and mixins come in.

In this post, we'll talk about how to build clean, maintainable, and scalable model architectures using Declarative Base, custom base classes, and mixins in SQLAlchemy. We'll start with the basics, build a solid foundation, and then move into more advanced usage patterns.

## Basics → What is an ORM?

An ORM (Object-Relational Mapper) is a way to interact with a database using Python objects. Instead of writing raw SQL queries, you define Python classes that map to database tables. SQLAlchemy is one of the most powerful and flexible ORMs in Python.

For example when you need to query for a user, you don’t say:

```sql
SELECT * FROM users WHERE id = 1;
```

Instead, you write:

```python
user = session.get(User, 1)
```

SQLAlchemy takes care of turning your Python objects into database rows and vice versa. And that’s huge. It means we can focus more on business logic and less on boilerplate SQL.

With SQLAlchemy, you can also define models like this:

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String)
```

Here, we:

- Define a base class using `declarative_base()`
- Create a `User` class with a table name and a few columns

This works fine for small apps, but what happens when you have dozens of models that all need the same fields like `id`, `created_at`, and `updated_at`?

## Basics → Declarative Base

SQLAlchemy needs a base class that all your models can inherit from. Think of it like the blueprint for all your tables.

```python
from sqlalchemy.orm import declarative_base

Base = declarative_base()
```

Now, when you write this:

```python
class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
```

SQLAlchemy understands that this class is a database table.

Why not just use SQLAlchemy’s default base? Because we often want custom behavior in all our models like timestamps, soft deletes, UUIDs, etc. So we create a custom base class and give it more power.

## Why Reusability Matters in ORMs

Imagine you’re working on a project with 20+ models. Most of them have:

- an `id` column
- timestamp fields like `created_at` and `updated_at`
- a `__tablename__`
- common behaviors like soft delete, UUID support, or audit logging

If you repeat this code everywhere, it’s hard to maintain. If you change one thing, you have to change it in 20 places. A better way is to centralize this logic and make it reusable.

---

Now, let’s build something real step by step!

## Step 1 → Create a Declarative Base

The Declarative Base is where all your models inherit from. Instead of using SQLAlchemy's default `Base`, we create our own with some shared logic:

```python
from sqlalchemy.orm import declarative_base

Base = declarative_base()
```

This base class becomes the foundation of every model in your project. It helps SQLAlchemy know which classes to register as database tables.

## Step 2 → Add a Custom Base Class with Shared Logic

Now let’s extend the base class by injecting common fields and behaviors:

```python
from sqlalchemy import Column, Integer, DateTime
from sqlalchemy.ext.declarative import declared_attr
import datetime

class CustomBase:
    id = Column(Integer, primary_key=True)

    @declared_attr
    def __tablename__(cls):
        return cls.__name__.lower()
```

- `id` is defined once and automatically included in all child models.
- `__tablename__` is generated dynamically based on the class name.

Now pass this class to the `declarative_base` factory:

```python
Base = declarative_base(cls=CustomBase)
```

From now on, any class that inherits from `Base` gets all these fields for free.

## Step 3 → Create Mixins for Optional Behavior

Mixins are reusable building blocks. They let you add specific features to a model only when you need them. Each mixin is just a class with some fields or methods.

### Timestamp Mixin

```python
class TimestampMixin:
    created_at = Column(DateTime, default=datetime.datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.datetime.utcnow, onupdate=datetime.datetime.utcnow)
```

Use this when you want to track creation and update times.

### Soft Delete Mixin

```python
class SoftDeleteMixin:
    deleted_at = Column(DateTime, nullable=True)

    def soft_delete(self):
        self.deleted_at = datetime.datetime.utcnow()
```

This lets you “delete” records without actually removing them from the database.

### UUID Primary Key Mixin

```python
import uuid
from sqlalchemy.dialects.postgresql import UUID

class UUIDMixin:
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
```

Use this when you want globally unique IDs instead of integers.

## Step 4 → Combine Base and Mixins

You can now create models like this:

```python
class User(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = 'users'

    name = Column(String, nullable=False)
    email = Column(String, nullable=False, unique=True)
```

This model now:

- has timestamps
- can be soft-deleted
- inherits shared logic from `Base`

## Step 5 → Audit Logging

In many production apps, you need to track who created or updated a record. You can do this with another mixin:

```python
class AuditMixin:
    created_by = Column(String)
    updated_by = Column(String)
```

Use it like this:

```python
class Invoice(Base, TimestampMixin, AuditMixin):
    __tablename__ = 'invoices'

    amount = Column(Float, nullable=False)
    description = Column(String)
```

This model now logs who made the last changes, great for admin dashboards and audit logs.

## Step 6 → Prevent Accidental Table Creation

Sometimes you want a mixin or base class that doesn’t create a table. You can mark it as abstract:

```python
from abc import ABC

class SoftDeleteBase(ABC):
    deleted_at = Column(DateTime)

class BaseSoftDeleteModel(Base, SoftDeleteBase):
    __abstract__ = True
```

SQLAlchemy skips abstract classes when creating tables.

## Alternative Usage → Declarative Base with Registry

Starting from SQLAlchemy 1.4, there's a newer and more modular way to define your models using a `registry()` object instead of directly calling `declarative_base()`. This approach provides greater flexibility, especially for larger codebases or plugin-style architectures where models are distributed across multiple files or packages.

```python
from sqlalchemy.orm import registry
from sqlalchemy.ext.declarative import declared_attr
from sqlalchemy import Column, Integer, String

mapper_registry = registry()

@mapper_registry.as_declarative_base()
class Base:
    @declared_attr
    def __tablename__(cls):
        return cls.__name__.lower()

    id = Column(Integer, primary_key=True)

```

Now, you can define your models by inheriting from `Base`:

```python
class Product(Base):
    name = Column(String, nullable=False)
    price = Column(Integer)
```

This is functionally similar to the classic `declarative_base()`, but gives you more control over how models are registered and mapped.

### Why Use This Over `declarative_base()`?

This approach is especially useful when:

- You're splitting your models into multiple modules.
- You want to use multiple metadata objects or control table registration.
- You're integrating SQLAlchemy into a plugin system or microservice architecture.
- You want to explicitly manage mapping configuration instead of relying on implicit global registration.

### Using `registry().map_imperatively()`

If needed, you can even mix in classic (imperative) mapping like this:

```python
from sqlalchemy import Table, Column, Integer, MetaData
from sqlalchemy.orm import registry

metadata = MetaData()
mapper_registry = registry(metadata=metadata)

user_table = Table(
    "users", metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String),
)

class User:
    pass

mapper_registry.map_imperatively(User, user_table)
```

This gives you full control over the mapping process and is handy in systems where models must be defined dynamically or loaded at runtime.

## Final Thoughts

Using Declarative Base and mixins isn’t just about saving time. It’s about:

- writing cleaner code
- making models easier to maintain
- avoiding duplication
- building a scalable system that grows with your project

These patterns work really well in large codebases and teams. Once you get the hang of it, your model layer becomes powerful, testable, and future-proof.