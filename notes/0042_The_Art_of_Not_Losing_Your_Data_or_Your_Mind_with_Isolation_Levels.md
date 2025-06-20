# The Art of Not Losing Your Data (or Your Mind) with Isolation Levels

Check this post on my blog [here](https://hevalhazalkurt.com/blog/the-art-of-not-losing-your-data-or-your-mind-with-isolation-levels/).

<br>

As backend developers, we like to think of our databases as these calm, rational entities, store a row here, update a value there, and life goes on. But once you throw in multiple users, parallel requests, and that dreaded word “concurrency” things can go sideways fast. This is where transaction isolation levels come into play. They’re like the traffic rules of your database: sometimes flexible, sometimes strict, but always essential if you want to avoid chaos. 

In this post, we’ll demystify isolation levels and show how different choices affect consistency, performance, and user experience. Ready? Let’s start!

## **Why Should We Care About Isolation Levels?**

Let’s say you’re building a ticket booking system. A user sees two tickets available for the upcoming Beyoncé concert (personally, I’d go for Manchester Orchestra but hey, it’s Beyoncé), and they rush to buy one. Meanwhile, another user tries to grab a ticket at the exact same moment. Now imagine: both users somehow end up buying the same ticket… or worse, the system tells both of them that they successfully booked the last ticket.

Yikes. We need to do something!

This is where transaction isolation levels come into play. They're like the traffic lights of the database world keeping things from crashing, colliding, or behaving unpredictably. And in this blog post, we’ll explore how databases manage concurrent operations, what can go wrong, and how to pick the right isolation level without losing your mind or your data.

But before we get fancy, let’s get the basics down.

### **What Is a Transaction Anyway?**

A transaction is like wrapping a bunch of database actions in a bubble that either completes fully or not at all.

In SQL, you usually see this:

```sql
BEGIN;
-- very important queries here
COMMIT;
```

In SQLAlchemy:

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    try:
        # you do cool stuff here
        session.commit()
    except:
        session.rollback()
        raise
```

### Why Bother With All That?

Because things can go wrong. Network issues, code bugs, or race conditions could break your flow. And when things go sideways, you don’t want to end up in a half-updated state where a user paid for a ticket but never actually got one.

Transactions follow something called **ACID**:

- **Atomicity →** all or nothing.
- **Consistency →** keep the data valid.
- **Isolation →** don’t mess with others’ transactions.
- **Durability →** once committed, it's permanent.

In this blog, we’re zoning in on the "I" (**Isolation)**.

### **What Is Isolation and Why Should You Care?**

Remember our case, two users trying to book a ticket at the same exact moment. Without proper isolation, their transactions could overlap in weird and dangerous ways. Isolation is all about making transactions think they’re the only one touching the database even if that’s not true.

Without it, you might see things like:

- one transaction reading data that another hasn't committed yet
- seeing inconsistent data because someone else changed it midway
- data just straight-up disappearing or reappearing like some SQL magic trick gone wrong

And trust me, you don’t want your app to be the magician in that trick. In the next section, we’ll dig into the different types of isolation levels (yes, there are more than one), and what each one actually does.

## **Isolation Levels (ANSI SQL Standard)**

Databases, much like humans, handle multitasking differently. Some are chill and let everyone do their thing; others are strict and micromanage every little move. In SQL terms, these “personalities” are known as Isolation Levels.

There are four main levels defined by the ANSI SQL standard:

### 1. **Read Uncommitted** → “YOLO Mode”

This is the least strict level. Transactions can read data that hasn't even been committed yet. Sounds dangerous, right? It is.

What can go wrong?

- You read dirty data.
- You make decisions based on things that might never exist because someone might roll back.

Example from our ticket system:

```sql
-- Transaction A
BEGIN;
UPDATE tickets SET status = 'reserved' WHERE id = 42;

-- Transaction B (at the same time)
SELECT status FROM tickets WHERE id = 42; 
-- Sees 'reserved', even though A hasn't committed
```

This can lead to “ghost reservations”, tickets that look reserved but actually aren’t. Users will rage. So, unless you’re doing analytics on logs or temp tables, avoid this like expired sushi.

### 2. **Read Committed** → “What You See Is What’s Been Done”

This is PostgreSQL’s default. A transaction can only read committed data, so no dirty reads. But it can still see changes if someone else commits mid-transaction.

What can go wrong?

- Non-repeatable reads → You read a row, someone updates it, and you read it again in the same transaction... and it’s different now.

Let’s look an example SQLAlchemy codes for our ticket system:

```python
with Session(engine) as session:
    ticket = session.execute(
        text("SELECT status FROM tickets WHERE id = 42")
    ).scalar_one()

    # Meanwhile another session updates this ticket

    ticket_again = session.execute(
        text("SELECT status FROM tickets WHERE id = 42")
    ).scalar_one()
```

You might end up with two different values within the same transaction. That’s fine for many apps but risky for stuff like payments or inventory. Use when → Your app can tolerate slightly stale data, but no dirty reads.

### 3. **Repeatable Read** → “What You See Is What You Always See”

Now we’re getting more serious. Once you read a row, it’s locked in memory for your transaction. You can read it 100 times, and it won’t change, even if someone else commits an update.

What can go wrong?

- Phantom reads → You do a `SELECT` that returns 3 rows, then someone inserts another row that would match your filter. You run the query again… now you get 4 rows. Great!

PostgreSQL handles this well using MVCC (multi-version concurrency control), so it won’t lock other users unnecessarily.

Let’s check available tickets for the concert:

```python
with Session(engine) as session:
    session.execute(text("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ"))

    result = session.execute(
        text("SELECT * FROM tickets WHERE event_id = 1 AND status = 'available'")
    ).fetchall()

    # Meanwhile another user cancel their tickets and a new available ticket in db now

    result_again = session.execute(
        text("SELECT * FROM tickets WHERE event_id = 1 AND status = 'available'")
    ).fetchall()

    assert result == result_again  
    # Still same 3 rows, even though new one was added
```

It’s great for: Inventory systems, ticket reservations, anything where “consistency” > “performance”.

### 4. **Serializable** → “Everyone Just Wait Their Turn”

This is the most strict level. It ensures that transactions happen as if they ran one after another; no overlap, no surprises. Sounds perfect, right? Well... it comes at a cost. More locking, more rollbacks, more “could not serialize access due to concurrent update” errors in your logs. 

Let’s look at another example: 

```python
with Session(engine) as session:
    session.execute(text("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE"))

    result = session.execute(
        text("SELECT * FROM tickets WHERE event_id = 1 AND status = 'available'")
    ).fetchall()

    # If another transaction is doing the same thing,
    # one of them might get rolled back.
```

Use when → You absolutely can’t afford anomalies like double-booking seats on a plane. Otherwise, it's probably overkill.

**Summary Table:**

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | PostgreSQL Support | Use Case |
| --- | --- | --- | --- | --- | --- |
| Read Uncommitted | ✅ | ✅ | ✅ | ❌ (PG uses Read Committed) | Never. Seriously, don’t! |
| Read Committed | ❌ | ✅ | ✅ | ✅ | General web apps |
| Repeatable Read | ❌ | ❌ | ✅ (handled well in PG) | ✅ | E-commerce, inventory, finance |
| Serializable | ❌ | ❌ | ❌ | ✅ (via predicate locking) | Strong consistency required |

## **Common Anomalies and What Causes Them**

Let’s dive into the kinds of weird stuff that can happen when isolation isn’t handled properly. These are the database horror stories you’re trying to avoid by picking the right isolation level.

### **Dirty Read** → Reading Someone Else’s Trash

A dirty read happens when you read data that another transaction hasn’t committed yet and might never commit at all.

The scenario (Have I mentioned before that I am/was a pro-filmmaker?):

- User A tries to reserve ticket #42, marks it as `reserved`.
- User B reads the ticket’s status before User A commits.
- Then User A gives up and rolls back.
- But User B has already made a decision based on a lie.

If you’re using PostgreSQL you’re lucky. It’s hard to do dirty reads in PostgreSQL because it doesn’t allow them. Good job, Postgres, I love you! But for other databases you need to fix it. You can use `Read Committed` level or higher.

### **Non-Repeatable Read** → When the Data Changes Mid-Sentence

You read a row once, then read it again later in the same transaction and it’s different. Maybe someone else updated it in between.

The Scenario:

- Your app loads the current status of ticket #42.
- You calculate how many tickets are left.
- Meanwhile, someone else cancels or updates a ticket.
- You read again and now the result is inconsistent.

To fix it you can use `Repeatable Read` or `Serializable`.

### **Phantom Read** → Data Appears (or Disappears) Out of Nowhere

You run a `SELECT` that returns N rows. Someone inserts a new row that would match your query. You run the same `SELECT` again and now you get N+1.

The Scenario:

- You check how many tickets are still available (`WHERE status = 'available'`).
- You get 3.
- Meanwhile, someone adds a new ticket.
- You query again in the same transaction. Now you get 4.

Again if you use PostgreSQL, its MVCC (Multi-Version Concurrency Control) handles this pretty well so in practice, phantom reads are rare in PG unless you're doing range queries or something really tricky. So, PostgreSQL's `Repeatable Read` is usually enough. But for other databases you need to fix it. You need to use `Serializable` if you're dealing with complex inserts or range conflicts. 

## So One Isolation Level to Rule Them All? Not Quite.

If there's one thing we can agree on after all this it’s that there’s no one-size-fits-all when it comes to isolation levels. Sure, it’s tempting to just crank everything up to `SERIALIZABLE` and call it a day. Maximum safety, right? But here’s the catch: your app isn’t a vault. It’s a ticket sales platform. Some parts need military-grade consistency… and some parts just need to move fast and not block users for no reason.

I mean, not every endpoint needs the same level of paranoia. Let’s go back to our ticketing system example:

**High-Stakes Flow → `POST /tickets/reserve`**

This endpoint actually reserves a ticket. Double booking? Huge no-no. We can’t afford race conditions here. So we need to sse `REPEATABLE READ` or even `SERIALIZABLE`, depending on how paranoid you are and how much load you expect.

**Read-Only Flow → `GET /events/{id}/tickets`**

This just lists available tickets. There’s no change, no commitment, just show me what’s there.So `READ COMMITTED` is totally fine here. You’ll get committed, real-enough data without any performance hits.

**Admin Reports → `GET /admin/revenue-summary`**

Again, you’re not writing anything. You might be reading slightly stale data, but unless your accountant is a ninja who notices 0.2 second differences, it doesn’t matter. Again, `READ COMMITTED` or even snapshot-based reads are perfect here.

### Smart Isolation = Better Performance + Safer Data

Using high isolation levels everywhere sounds safe but it’s actually risky:

- Slower queries
- More locks
- Higher chance of deadlocks
- Angry users staring at a loading spinner

Instead, think in tiers:

- Critical paths like payments, reservations, inventory adjustments: Go stricter.
- Read-heavy endpoints like search, browse, analytics: Keep it light.

This layered approach gives you the best of both worlds:

- Your data stays consistent where it matters.
- Your users stay happy where speed matters.

## Final Words

Transactions are powerful. Isolation levels give you control. But like any power tool, if you just smash everything with the same setting, you’re gonna have a bad time. Pick the right tool for the job.