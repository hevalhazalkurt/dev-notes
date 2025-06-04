# Optimistic vs. Pessimistic Locking in ORMs

Check this post on my blog [here](https://hevalhazalkurt.com/blog/optimistic-vs-pessimistic-locking-in-orms/).

<br>


If you've ever worked on a web app with a database, you've probably run into weird bugs when two users try to update the same thing at the same time. Sometimes one user’s changes mysteriously disappear. Other times, the app crashes with vague errors about stale data or database locks.

These problems are all about concurrent data access, and ORMs like SQLAlchemy, Django ORM, or Hibernate, try to help you deal with it using locking strategies. The two main ones are:

- **Optimistic Locking →** “Let’s hope no one else updates this before me.”
- **Pessimistic Locking → “**I’m going to lock this row so no one else can mess with it.”

This post is a deep dive into how these two strategies work in ORMs, what can go wrong, and how you can build more robust systems by understanding the mechanics under the hood.

## The Basic Idea Behind Locking

In a multi-user system, you can’t assume you're the only one updating the data. Locking is a way to manage who gets to update what, and when.

### Optimistic Locking

Optimistic locking assumes that most of the time, no one else will touch the same data you're working on. So you:

1. Read the data
2. Make your changes
3. Try to save
4. If someone else changed it first, you get an error, and you need to retry.

This is often done with a version column (like `version` or `updated_at`).

### Pessimistic Locking

Pessimistic locking assumes conflict is likely, so it prevents others from touching the data until you're done. You tell the database:

“Lock this row — no one else can update or even read it until I’m finished.”

This happens with SQL’s `SELECT ... FOR UPDATE` or its variants like `FOR UPDATE SKIP LOCKED`.

## Understanding Lock Types in SQL and ORMs

In PostgreSQL and most relational DBs, locks can happen at different levels:

- **Row-level locks →** Prevent concurrent changes to specific rows.
- **Table-level locks →** Affect the entire table. It’s rare in most ORM apps.
- **Advisory locks →** App-defined, not tied to actual data.

We'll focus on row-level locks, since they're what ORMs like SQLAlchemy and Django use when you do `FOR UPDATE`.

## Row-Level Lock Types in PostgreSQL

| **Lock Type** | **Behavior** | **Common Use** |
| --- | --- | --- |
| `FOR UPDATE` | Locks row for update | Safe updates by one user only |
| `FOR NO KEY UPDATE` | Like `FOR UPDATE`, but weaker | Avoid locking foreign keys |
| `FOR SHARE` | Other readers allowed, no updates | Long reads, analysis |
| `FOR KEY SHARE` | Others can read but not update keys | Used with foreign keys |
| `SKIP LOCKED` | Skip locked rows instead of waiting | Job queues |
| `NOWAIT` | Raise error if row is already locked | Immediate fail if contention |

Let’s go over these with real examples.

### `FOR UPDATE`

This is the most common and strict lock. It prevents anyone else from updating or deleting the locked row.

```python
session.query(User)
    .filter(User.id == 1)
    .with_for_update()
    .one()
```

Use this when:

- You need to read-update-write safely.
- Example: Editing a user profile and saving changes.

### `FOR UPDATE SKIP LOCKED`

Instead of waiting for locked rows to be free, this just skips them. This is gold for worker queues.

```python
session.query(Job)
    .filter(Job.status == 'pending')
    .order_by(Job.created_at)
    .with_for_update(skip_locked=True)
    .limit(1)
    .all()
```

Use this when:

- You’re building a background job system.
- You want one worker to take a job, and others to move on.

This prevents double-processing and deadlocks.

### `FOR UPDATE NOWAIT`

This one throws an error right away if someone else has the row locked.

```python
session.query(Product)
    .filter(Product.id == 42)
    .with_for_update(nowait=True)
    .one()
```

Use this when:

- You want immediate feedback that a row is locked.
- Great in real-time systems where waiting is not okay like stock trading.

### `FOR SHARE`

This allows others to also read (share) the row, but prevents updates. You can use it with PostgreSQL raw SQL or certain advanced ORMs that support it.

Use this for:

- Reporting systems or analytics that need data to be consistent but not block writers.

```sql
SELECT * FROM users WHERE id = 1 FOR SHARE;
```

### `FOR NO KEY UPDATE` and `FOR KEY SHARE`

These are rarely used directly in app-level code, but they matter if you're working with foreign keys or complex relationships.

- `FOR NO KEY UPDATE`: Prevents update/delete of row, but not FK changes.
- `FOR KEY SHARE`: Prevents FK from pointing to a deleted row.

In SQLAlchemy you probably won’t use these explicitly unless you’re doing complex cascading logic or dealing with graph-like data.

## What It Looks Like in ORMs

Let’s explore both strategies using SQLAlchemy as the example, but the same logic applies to Django ORM or Hibernate.

## Optimistic Locking with SQLAlchemy

### Pattern → Version Column

Add a `version` column to your table:

```python
from sqlalchemy.orm import declarative_base, versioned

Base = declarative_base()

class Document(Base):
    __tablename__ = 'documents'
    id = Column(Integer, primary_key=True)
    content = Column(Text)
    version = Column(Integer, nullable=False, default=1)

    __mapper_args__ = {
        "version_id_col": version
    }
```

### How It Works

SQLAlchemy will:

- Include the current `version` in the `WHERE` clause when updating
- If another update already bumped the version, the update fails (`0 rows affected`)
- It raises a `StaleDataError`

### Common Pitfalls

1. **Not retrying on failure**
    
    Many devs just log the error. Instead, you should show the user a warning, reload data, and offer a retry or merge.
    
2. **No version column = no conflict detection**
    
    ORMs can't protect you without a `version` field or similar mechanism.
    
3. **Silent overwrite in Django/Hibernate if not configured**
    
    Without explicit version tracking, updates just overwrite each other with no warning.
    

## Pessimistic Locking with SQLAlchemy

### Pattern → `FOR UPDATE`

Use `with_for_update()` to lock rows during queries:

```python
session = Session()

doc = session.query(Document)
    .filter_by(id=1)
    .with_for_update()
    .one()
```

Now that row is locked, nobody else can select it with a lock, and any update must wait for your transaction to finish.

### Real-World Use Case

You're writing a job queue:

- You want one worker to pick a job
- Others should skip it if someone already picked it

```python
session.query(Job)
    .filter(Job.status == 'pending')
    .order_by(Job.created_at)
    .with_for_update(skip_locked=True)
    .limit(5)
```

### Common Pitfalls

1. **Forgetting to commit/rollback**
    
    If your transaction hangs, the row stays locked = deadlocks and stuck workers.
    
2. **Locking too many rows**
    
    Locking large sets can kill performance. Always limit.
    
3. **Missing indexes**
    
    Locking with `ORDER BY` needs proper indexes to avoid full table scans.
    

## ORM-Specific Gotchas

### SQLAlchemy

- Works great with both types of locking.
- Be careful with session scoping in async systems like FastAPI.
- `with_for_update()` is explicit. You won’t lock anything unless you ask.

### Django ORM

- No built-in optimistic locking unless you write it yourself.
- Supports `select_for_update()` similar to SQLAlchemy.

## When to Use Which?

| **Situation** | **Use** |
| --- | --- |
| Most updates don’t conflict, but correctness matters | Optimistic Locking |
| Heavy write contention (e.g. same row in background jobs) | Pessimistic Locking |
| Long-running user edits (like forms, drafts) | Optimistic Locking with retries |
| Critical operations (money, inventory) | Pessimistic Locking to avoid risk |

## Final Thoughts

Locking isn’t just a low-level database thing, it’s a design decision. ORMs can help, but only if you understand how they work. 

Optimistic locking is great for user-facing apps where you want fast performance and low chance of conflict.

Pessimistic locking shines in backend systems where correctness matters more than speed like job queues, banking, and inventory systems.

Your ORM is only as smart as you configure it. Make sure you pick the right strategy based on your real-world needs and test for concurrency issues early.