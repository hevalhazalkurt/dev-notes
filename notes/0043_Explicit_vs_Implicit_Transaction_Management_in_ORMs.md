# Explicit vs Implicit Transaction Management in ORMs

Check this post on my blog [here](https://hevalhazalkurt.com/blog/explicit-vs-implicit-transaction-management-in-orms/).

<br>

If you're a backend developer working with SQLAlchemy, you've likely come across transactions in two forms: sometimes they just “work” (thanks to the ORM), and other times, you're told to explicitly begin, commit, or rollback. Most tutorials gloss over the difference between implicit and explicit transaction management, but in real-world systems, especially under load or failure, this distinction can be the difference between reliable data and a corrupted database.

In this post, we’ll explore what happens behind the scenes of SQLAlchemy’s transaction management, where implicit handling can betray you, and how to take control explicitly. We’ll walk through examples from backend development scenarios like API requests, background jobs, and error-prone workflows to show when implicit transactions fall short.

## Understanding the Basics

### What Is a Transaction?

A transaction is a sequence of database operations that must either complete fully or not at all. In PostgreSQL, MySQL, and other RDBMSs, transactions guarantee ACID properties:

- Atomicity → All or nothing.
- Consistency → The DB moves from one valid state to another.
- Isolation → Intermediate states are invisible to others.
- Durability → Once committed, it stays committed.

In SQLAlchemy, you can manage these transactions manually (explicit) or let the ORM do it (implicit).

### Implicit Transactions in SQLAlchemy ORM

By default, SQLAlchemy ORM will open and manage a transaction implicitly behind the scenes when you use a session. Let’s look at an example:

```python
from sqlalchemy.orm import Session
from models import User  # assume a simple User model

def create_user(engine, username):
    session = Session(engine)
    user = User(username=username)
    session.add(user)
    session.commit()
```

Here’s what’s happening under the hood:

- `Session(engine)` begins a new transaction.
- `session.add()` queues the insert.
- `session.commit()` commits the transaction.

Pretty clean, right? But what if something goes wrong before `commit()`?

### The First Pitfall → Unhandled Exceptions

Consider this modified version:

```python
def create_user(engine, username):
    session = Session(engine)
    user = User(username=username)
    session.add(user)

    # Simulate a crash
    raise ValueError("Unexpected error before commit")

    session.commit()
```

Even though the user was added, the session is left open and the transaction is still active. If you reuse this session or forget to call `rollback()`, you could hit confusing behavior like getting stuck in a bad state or locking rows longer than expected.

**Why is this dangerous?**

- You may leak locks in the DB.
- Future operations using this session will fail until you rollback.
- You’ve left partial state hanging.

In short → Implicit transaction management works until it doesn’t, especially under exceptions.

### Explicit Transaction Management with Context Managers

To avoid these problems, you can manage transactions explicitly using a context manager:

```python
from sqlalchemy.orm import Session

def create_user(engine, username):
    with Session(engine) as session:
        try:
            user = User(username=username)
            session.add(user)
            session.commit()
        except Exception as e:
            session.rollback()
            raise
```

Here:

- `with Session(engine)` ensures the session is cleaned up.
- The `try/except` block ensures rollback is called on error.

This is the recommended pattern for web applications, background jobs, or anything production-grade.

## Common Anti-patterns in Real Life

Let’s look at an example from a backend API endpoint:

```python
@app.post("/users")
def register_user(username: str):
    session = Session(engine)  # no context, no rollback
    user = User(username=username)
    session.add(user)
    if username == "crash":
        raise Exception("Boom!")
    session.commit()
```

This code is functionally correct until an exception happens. Then:

- Session is left unclosed.
- Transaction is left hanging.
- You may hit DB-level locks or transaction limit errors.

Now compare to the explicit version:

```python
@app.post("/users")
def register_user(username: str):
    with Session(engine) as session:
        try:
            user = User(username=username)
            session.add(user)
            if username == "crash":
                raise Exception("Boom!")
            session.commit()
        except:
            session.rollback()
            raise
```

This handles both cleanup and failure scenarios predictably.

## Savepoints and Nested Transactions in SQLAlchemy

Imagine a backend workflow like this:

> Create a user, assign them to a group, and send a welcome notification, but if notification fails, we still want the user and group assignment to succeed.
> 

This is a good candidate for nested transactions, also known as savepoints.

### What Is a Savepoint?

A savepoint allows you to rollback to a specific point within a transaction without discarding the whole thing. This is useful when part of your logic is optional, failure-prone, or slow third-party integrations. Let’s look at that SQLAlchemy example:

```python
from sqlalchemy.orm import Session
from sqlalchemy.exc import SQLAlchemyError

def onboard_user(engine, username):
    with Session(engine) as session:
        try:
            user = User(username=username)
            session.add(user)
            session.flush()  # assign user.id before savepoint

            # Assign group
            group = Group(name="default")
            user.groups.append(group)
            session.flush()

            # Savepoint for optional step
            with session.begin_nested():
                try:
                    send_welcome_email(user.email)  # may raise exception
                except EmailServiceException:
                    print("Failed to send email, rolling back optional step.")
                    # rollback only to savepoint

            session.commit()

        except SQLAlchemyError:
            session.rollback()
            raise
```

So what’s happening here:

- `session.begin_nested()` creates a savepoint
- If email fails, only that part is rolled back, not the entire transaction
- `session.flush()` is used to push changes to DB but not commit so IDs are available.

This pattern is especially useful when integrating with flaky external services.

## Handling Partial Failures in Batch Jobs

Imagine you're importing 1000 users from a CSV file. Some may fail maybe because of bad data, but you don’t want one error to block the entire import.

### Anti-pattern → One Big Transaction

```python
with Session(engine) as session:
    for row in csv_rows:
        user = User(username=row['username'])
        session.add(user)
    session.commit()  # risky if any row fails
```

If one row causes an exception, everything rolls back.

### Better → Commit in Chunks, Use Savepoints

```python
with Session(engine) as session:
    for row in csv_rows:
        try:
            with session.begin_nested():  # savepoint per row
                user = User(username=row['username'])
                session.add(user)
        except Exception as e:
            print(f"Failed to insert {row['username']}: {e}")

    session.commit()  # commit what succeeded
```

This way:

- Each row is isolated.
- Only bad rows are skipped.
- You still get a single DB commit at the end.

## Retryable Transactions: Dealing with Deadlocks or Transient Failures

Some failures like serialization errors or deadlocks are retryable. You should catch and retry the whole transaction. Let’s look at a little retrying flow on serialization failure.

```python
from sqlalchemy.exc import OperationalError
import time

def with_retry(max_retries=3):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except OperationalError as e:
                    if 'could not serialize access' in str(e):
                        print(f"Retry {attempt+1} due to concurrency error")
                        time.sleep(0.5)
                    else:
                        raise
            raise RuntimeError("Too many retries")
        return wrapper
    return decorator

@with_retry()
def transfer_funds(engine, from_id, to_id, amount):
    with Session(engine) as session:
        sender = session.get(Account, from_id)
        receiver = session.get(Account, to_id)
        sender.balance -= amount
        receiver.balance += amount
        session.commit()
```

This retry logic only works if the whole transaction block is safe to retry. Be careful with side-effects like emails or logs inside retries.

## Async Transactions with SQLAlchemy 2.0

### Why This Matters

Modern Python web frameworks like FastAPI and async job runners like Celery with green threads are pushing developers toward `asyncio`. SQLAlchemy 2.0 introduced native async support but transaction semantics are slightly different.

Let’s explore the basic async example below.

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def create_user_async(username):
    async with async_session() as session:
        async with session.begin():
            user = User(username=username)
            session.add(user)
```

Note the double async with → one for the session, another for the transaction. This ensures that both the session and its transaction are explicitly managed and correctly cleaned up.

### No Implicit Transactions in Async

Unlike the sync API, the async API does not start a transaction on session creation, you must use `session.begin()` explicitly, or your writes won’t be committed.

That means you can’t rely on implicit commits anymore in async code. This is a good thing, but it catches many people off guard when migrating.

## ORM vs Core Transaction Differences

Sometimes you’ll drop into SQLAlchemy Core to write raw SQL or bulk operations. But beware the transaction behavior is subtly different.

### ORM Session Transaction Model

- Starts a transaction when needed (on first write).
- Commits or rolls back with `session.commit()` or `session.rollback()`.
- Knows about ORM unit-of-work lifecycle.

### Core Connection Transaction Model

- You manually call `begin()`, `commit()`, `rollback()` on `Connection`.
- No unit-of-work.
- No flush/expire logic.

Example:

```python
from sqlalchemy import text

with engine.begin() as conn:
    conn.execute(text("INSERT INTO audit_log (message) VALUES (:msg)"), {"msg": "manual insert"})
```

If you mix Core and ORM, be aware that they don’t share the same transactional context unless you wire it up manually.

## Diagnosing Hanging or Long-Running Transactions

Nothing kills DB performance like a rogue transaction that forgot to `commit` or `rollback`. Let’s walk through how to detect them.

### 1. Use Database-Level Inspection

For PostgreSQL:

```sql
SELECT pid, state, query, xact_start, backend_start
FROM pg_stat_activity
WHERE state = 'idle in transaction';
```

This shows sessions that are *open and waiting*, often due to a missing commit/rollback.

### 2. Use SQLAlchemy Event Hooks

SQLAlchemy lets you hook into session and transaction lifecycle events to log, time, or trace transaction durations.

```python
from sqlalchemy import event
from sqlalchemy.orm import Session
import time

@event.listens_for(Session, "after_begin")
def track_start(session, transaction, connection):
    session.info['tx_start'] = time.time()

@event.listens_for(Session, "after_transaction_end")
def track_end(session, transaction):
    start = session.info.pop('tx_start', None)
    if start:
        duration = time.time() - start
        print(f"Transaction took {duration:.2f}s")
```

This is invaluable for catching long-running or zombie transactions early.

## In The End Transactions Are Invisible Until They Fail

As backend developers, we often take transactions for granted. We write a few `session.add()` lines, maybe a `session.commit()`, and assume everything just works. But under the hood, the way transactions are managed, explicitly or implicitly, has a massive impact on data consistency, system behavior under failure, and your ability to debug real-world issues.

Let it be this: transaction boundaries should be a conscious design decision, not an accident of framework behavior. Just like you wouldn’t let your API endpoints return random status codes or log messages to nowhere, you shouldn’t let your DB writes rely on invisible, implicit, or uncontrolled transaction flows.

Make it explicit. Make it observable. Make it testable.

Because when production data is on the line especially at scale “I thought SQLAlchemy handled that for me” is not a great incident report.