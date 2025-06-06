# Handling Data in Alembic Migrations When Schema Changes Aren’t Enough

Check this post on my blog [here](https://hevalhazalkurt.com/blog/handling-data-in-alembic-migrations-when-schema-changes-arent-enough/).

<br>

In most backend projects, database migrations are treated as structural like adding a column here, renaming a table there, dropping an index, and so on. These are the kinds of changes your migration tool like Alembic for SQLAlchemy or Django's migration system handles really well. But what happens when just changing the schema isn't enough?

What if the data itself needs to change along with the schema?

What if you're adding a non-nullable column but need to backfill historical values?

Or updating enum types?

Or normalizing denormalized legacy fields?

This is where data-backed migrations come in.

In this post, we’ll cover:

- What data-backed migrations are and why they matter
- When and how to write them
- Real-world backend use cases
- Pitfalls and best practices
- How to keep them safe and maintainable

Let’s get into it.

## What Is a Data-Backed Migration?

A data-backed migration is a database migration that modifies existing data as part of a structural schema change. Most schema changes like `ALTER TABLE` don’t touch any rows. They just change how the table is defined. But sometimes, after you make a schema change, your existing data no longer fits the new structure, or you want to prepare it for the new structure. That’s where you write custom logic to update the data inside your migration script, usually using SQL or your ORM.

### Common Scenarios

Here are some common use cases:

| **Use Case** | **Example** |
| --- | --- |
| New column with historical values | Add `is_active` column and set it to `true` for all current users |
| Enum updates | Rename enum values in existing rows |
| JSON refactoring | Change nested keys in JSON fields |
| Data normalization | Move comma-separated values into a new association table |
| Soft delete | Convert a `deleted_at IS NULL` pattern into a proper `is_deleted` boolean |

## Why Not Just Use a Script?

You might be thinking: “Can’t I just run a one-time Python or SQL script to fix the data?” Yes, but that means you're bypassing your migration history. The main advantage of doing it inside a migration is versioning and repeatability. Everyone on your team and your CI/CD pipeline can apply the same transformation consistently. Plus, one-off scripts often get lost. Migrations don’t.

## Writing a Simple Data-Backed Migration

Let’s say we want to add a new column called `is_premium` to a `users` table and set it to `True` for any user with more than 10 orders.

### Step 1 → Schema Change

In an Alembic migration:

```python
op.add_column('users', sa.Column('is_premium', sa.Boolean(), nullable=True))
```

If we stop here, the new column will exist, but it’ll be empty (`NULL`) for all existing users.

### Step 2 → Data Backfill

Here’s how we update the data inside the same migration:

```python
from sqlalchemy.sql import table, column, text
from sqlalchemy import Boolean, Integer

def upgrade():
    # Step 1: Add the column
    op.add_column('users', sa.Column('is_premium', sa.Boolean(), nullable=True))

    # Step 2: Fill in existing data
    op.execute("""
        UPDATE users
        SET is_premium = TRUE
        WHERE id IN (
            SELECT user_id FROM orders
            GROUP BY user_id
            HAVING COUNT(*) > 10
        )
    """)

    # Step 3: Make it NOT NULL (optional)
    op.alter_column('users', 'is_premium', nullable=False, server_default='FALSE')
```

Now when someone runs this migration, the schema and the data change *together*. Future developers won’t be confused. Everyone stays in sync.

## Advanced Use Cases

### 1. **Enum Value Changes**

Dealing with PostgreSQL enums during migrations is often a headache. The experience is talking. Unlike standard columns, enums are part of the database type system, which means:

- You can't drop an enum type if a column still uses it.
- Changing enum values like renaming or adding options usually requires intermediate steps.
- Adding new enum values is even more harder.
- Enums are cached in PostgreSQL, so you must be very deliberate when altering them.

Because of this, it's important to handle enum migrations safely and repeatably.

Let’s say we want to change the values in an existing PostgreSQL enum field called `status`, used in a `tasks` table like below.

Original Enum: `'todo'`, `'in_progress'`, `'done'`

New Enum: `'pending'`, `'active'`, `'complete'`

Mapping:

- `'todo' → 'pending'`
- `'in_progress' → 'active'`
- `'done' → 'complete'`

**The Hard Way (Manually):** You can do all these steps manually (as shown earlier), but this becomes error-prone if you do it often or across multiple tables/columns.

```sql
ALTER TYPE status RENAME VALUE 'todo' TO 'pending';
```

And if that fails on Postgres < 10, you might need to:

- Create a new enum
- Update rows to use the new values
- Alter the column
- Drop the old type

Yes, it’s messy, but that's real life. And you should keep it versioned in your migrations.

So instead...

**Reusable `upgrade_enum` Helper:** We can abstract all the steps into a reusable function like below.

```python
def upgrade_enum(
    op,
    table: str,
    column: str,
    enum_name: str,
    old_options: list[str],
    new_options: list[str],
    old_to_new_mapping: dict[str, str] | None = None,
):
    tmp_name = f"tmp_{enum_name}"
    tmp_options = set([*old_options, *new_options])

    old_type = sa.Enum(*old_options, name=enum_name)
    new_type = sa.Enum(*new_options, name=enum_name)
    tmp_type = sa.Enum(*tmp_options, name=tmp_name)

    # Step 1: Create a temporary enum type with all options
    tmp_type.create(op.get_bind(), checkfirst=False)

    # Step 2: Alter the column to use the temporary enum
    op.execute(
        f"ALTER TABLE {table} ALTER COLUMN {column} TYPE {tmp_name} USING {column}::text::{tmp_name}"
    )

    # Step 3: Optionally convert old values to new values
    if old_to_new_mapping:
        for old, new in old_to_new_mapping.items():
            op.execute(f"UPDATE {table} SET {column}='{new}' WHERE {column}='{old}'")

    # Step 4: Drop old enum type and create new one
    old_type.drop(op.get_bind(), checkfirst=False)
    new_type.create(op.get_bind(), checkfirst=False)

    # Step 5: Migrate from temporary enum to new enum
    op.execute(
        f"ALTER TABLE {table} ALTER COLUMN {column} TYPE {enum_name} USING {column}::text::{enum_name}"
    )

    # Step 6: Drop the temporary enum type
    tmp_type.drop(op.get_bind(), checkfirst=False)
```

Don’t forget, you also need to create **`downgrade_enum`** function.

Now we can change our enums easily!

```python
from alembic import op
import sqlalchemy as sa
from myapp.utils import upgrade_enum  

def upgrade():
    upgrade_enum(
        op=op,
        table="tasks",
        column="status",
        enum_name="status",
        old_options=["todo", "in_progress", "done"],
        new_options=["pending", "active", "complete"],
        old_to_new_mapping={
            "todo": "pending",
            "in_progress": "active",
            "done": "complete"
        }
    )
```

### 2. **Splitting JSON Fields**

Suppose you have a `settings` JSON field in the `users` table like this:

```json
{
  "email_notifications": true,
  "dark_mode": false
}
```

And now you want to split these into dedicated columns: `email_notifications`, `dark_mode`.

In your migration:

```python
op.add_column('users', sa.Column('email_notifications', sa.Boolean()))
op.add_column('users', sa.Column('dark_mode', sa.Boolean()))

op.execute("""
    UPDATE users
    SET email_notifications = (settings->>'email_notifications')::boolean,
        dark_mode = (settings->>'dark_mode')::boolean
""")
```

You just extracted structured data from semi-structured JSON and migrated it into proper schema. That’s exactly what data-backed migrations are for.

### 3. **Denormalization or Re-architecture**

Let’s say you used to store tags as a comma-separated string:

```python
blog_posts.tags = "python,backend,orm"
```

Now you want to normalize them into a `post_tags` association table.

You’d:

- Create the new table
- Backfill the tags
- Remove the old column

That backfill logic needs custom Python (not just SQL). You might write a helper in your migration that:

1. Reads each blog post
2. Splits the tags
3. Inserts into `post_tags` with proper FK references

Yes, it’s slower than raw SQL, but it ensures consistency. 

## Pitfalls & Best Practices

### Avoid Long-Running Data Updates

Never scan millions of rows in a single transaction during a migration. You risk:

- Locking tables
- Transaction timeouts
- Failed deployments

Use batching if needed like loop over 10k rows per commit

### Keep Migrations Idempotent Where Possible

If your migration is re-run, will it break things? If yes, that’s risky.

For example:

```sql
UPDATE users SET role = 'admin' WHERE role = 'superuser'
```

What happens if you run this again? Nothing bad, but it’s safe because the condition limits it.

- Always add `WHERE` clauses
- Use `INSERT ... ON CONFLICT DO NOTHING`
- Avoid hard deletes unless you’re *sure*

### Use Raw SQL Where Needed, ORM Where Safe

- Use raw SQL for performance-critical updates
- Use ORM logic if you need Python-side logic like parsing, serialization

## Final Thoughts

Data-backed migrations are the real workhorses of a mature backend system.

They're:

- Safer than running one-off scripts
- More maintainable than ad hoc changes
- Essential when schema changes depend on data or vice versa

They require more thought and care, but they give you consistency, traceability, and confidence in return. So the next time you're adding a column, updating enums, or reworking relationships ask yourself: “Does the data need to evolve, too?” If yes, it’s time for a data-backed migration.