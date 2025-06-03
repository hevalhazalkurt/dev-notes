# Designing Robust Transaction Management with Nested Transactions and Savepoints in SQLAlchemy

Check this post on my blog [here](https://hevalhazalkurt.com/blog/designing-robust-transaction-management-with-nested-transactions-and-savepoints-in-sqlalchemy/).

<br>

When you’re building real-world backend systems, especially ones that handle money, inventory, or other sensitive data, you can't afford to get transactions wrong. SQLAlchemy gives you powerful tools to manage transactions properly. But while `session.commit()` and `session.rollback()` are familiar to most developers, concepts like nested transactions, savepoints, and session state are where real control lives.

In this blog post, we’ll start with the basics, then dig deeper into how to manage complex transaction flows using SQLAlchemy. We’ll use backend examples to make it easy to follow.

## What Is a Transaction, Really?

A transaction is a unit of work in a database that is:

- **Atomic**: Everything in the transaction succeeds, or none of it does.
- **Consistent**: The database remains valid after the transaction.
- **Isolated**: Transactions don't interfere with each other.
- **Durable**: Once committed, changes are saved even if the server crashes.

In SQLAlchemy, a session manages the transaction lifecycle for you:

```python
session = Session()
try:
    # Perform some operations
    session.commit()
except:
    session.rollback()
finally:
    session.close()
```

That’s fine for simple things. But what if you need to partially roll back? Or catch an error, fix something, and try again without starting the entire transaction over? That’s where advanced transaction control comes in.

## The Basics → `commit()` and `rollback()`

Let’s say you’re processing a user’s checkout order:

```python
def process_checkout(session, user_id, items):
    user = session.get(User, user_id)
    order = Order(user_id=user.id)
    session.add(order)

    for item in items:
        line = OrderLine(order=order, item_id=item.id, quantity=item.qty)
        session.add(line)

    session.commit()
```

If something goes wrong like an item is out of stock, you’ll want to roll everything back.

```python
try:
    process_checkout(session, user_id, cart_items)
except Exception as e:
    session.rollback()
    log_error(e)
```

That’s fine for all-or-nothing scenarios. But sometimes, we need more control.

## What Are Savepoints and Nested Transactions?

Imagine this flow in a finance app:

- Start a transaction.
- Add a user payment.
- Log the payment.
- Call a third-party payment API.
- If the API fails, rollback only the API call, but keep the log and user record.

A full rollback loses everything. A savepoint lets you rewind to just before the API call, fix it, and try again. SQLAlchemy supports this using nested transactions, which use database savepoints under the hood.

## Using Nested Transactions with `session.begin_nested()`

Here’s an example:

```python
def process_payment(session, user_id, amount):
    user = session.get(User, user_id)
    session.add(PaymentLog(user_id=user.id, amount=amount))

    nested = session.begin_nested()
    try:
        result = call_payment_api(user_id, amount)
        session.add(Payment(user_id=user.id, confirmation=result.id))
        nested.commit()
    except PaymentAPIError:
        nested.rollback()  # Only rollback the payment part
        notify_support(user_id)

    session.commit()
```

So, what’s happening here? 

- The outer session starts when you call `session.begin()` or use a sessionmaker.
- The nested transaction creates a savepoint.
- If `call_payment_api()` fails, we roll back just the part inside the savepoint.
- The outer transaction like logging is still safe to commit.

This is gold for systems where partial progress is valid like logging, audit trails, or user-notified errors.

## Retry Logic with Nested Transactions

Let’s say the API is flaky and you want to retry:

```python
for _ in range(3):
    nested = session.begin_nested()
    try:
        confirmation = call_payment_api(user_id, amount)
        session.add(Payment(user_id=user.id, confirmation=confirmation.id))
        nested.commit()
        break
    except PaymentAPIError:
        nested.rollback()
else:
    raise PaymentAPIError("Failed after 3 attempts")
```

And as a result,

- You don’t lose the full session
- You only retry the part that failed
- Clean, isolated, and no weird states

## Example → Batch Data Imports

Imagine importing 1,000 rows of user-submitted data. You don’t want the whole job to fail because one row was malformed.

```python
def import_data(session, rows):
    for row in rows:
        nested = session.begin_nested()
        try:
            user = User(name=row['name'], email=row['email'])
            session.add(user)
            nested.commit()
        except Exception as e:
            nested.rollback()
            log_invalid_row(row, str(e))
    session.commit()
```

This lets you →

- Save valid rows
- Skip bad ones
- Commit all the good stuff at once

## Under the Hood → Session States

To really master this stuff, you need to understand session state transitions:

| **Action** | **Effect** |
| --- | --- |
| `session.add(obj)` | Marks object as “new” (pending) |
| `session.commit()` | Flushes changes, starts new transaction |
| `session.rollback()` | Reverts all uncommitted changes |
| `session.begin_nested()` | Creates a savepoint (starts nested tx) |
| `session.flush()` | Sends pending changes to the DB (without committing) |

Use `.flush()` if you want the DB to raise constraints early, before `commit()`.

## Best Practices for Transaction Management

### 1. Always Use `try/except/finally`

Don’t rely on implicit rollback. Explicit is better than implicit.

```python
try:
    session.begin()
    # do work
    session.commit()
except:
    session.rollback()
    raise
finally:
    session.close()
```

### 2. Test Your Transaction Boundaries

Use tests to simulate partial failures and assert what remains committed.

### 3. Use Savepoints for Retry-Safe Logic

Especially for flaky APIs, partial inserts, or user-initiated recovery steps.

### 4. Keep Transactions Short

Long-running transactions can block rows, cause deadlocks, and hurt performance. Break up large jobs into safe chunks.

## Final Thoughts

Mastering transaction management in SQLAlchemy is one of the biggest upgrades you can make in your backend development skills. Whether you're building a fintech app, logistics system, or high-concurrency SaaS API, knowing how to properly use savepoints, nested transactions, and session control will make your app safer, more scalable, and easier to debug.