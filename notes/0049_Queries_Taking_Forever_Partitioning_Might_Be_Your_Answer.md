# Queries Taking Forever? Partitioning Might Be Your Answer

Check this post on my blog [here](https://hevalhazalkurt.com/blog/queries-taking-forever-partitioning-might-be-your-answer/).

<br>


Alright, let's talk. You're a Python backend developer. You've built a slick e-commerce platform, “PyShop”, using FastAPI or Django, with your trusty friend SQLAlchemy handling the ORM duties. Everything is great. Users are signing up, orders are pouring in. Life is good.

Then, one day, you get a ticket: “The monthly sales report is timing out”. You `ssh` into the server, pop open psql, and run a `SELECT COUNT(*) FROM orders`;. The number that comes back is… large. Terrifyingly large. Every query with a `WHERE created_at BETWEEN ...` clause now takes an eternity. Your orders table has become a monolithic beast, and it's starting to slow everything down.

We've all been there. This is the moment when you stop thinking about new features and start thinking about infrastructure. And one of the most powerful tools in your PostgreSQL arsenal for this exact problem is Table Partitioning.

In this guide, we'll take a deep dive into PostgreSQL partitioning from a Python developer's perspective. We'll ditch the dry, academic definitions and use our struggling PyShop e-commerce platform to see how to implement this in the real world, using SQLAlchemy and a bit of Python scripting magic.

## Let’s Start With Basics → What is Partitioning?

At its core, partitioning is brilliantly simple. Imagine your orders table is a single, massive filing cabinet drawer. To find an order from January 2022, you have to rummage through the entire drawer, which also contains orders from 2021, 2023, and so on. It's a mess. Partitioning is like replacing that one giant drawer with a proper filing cabinet.

You create a “logical” or “parent” table (`orders`) that you still query as a single entity. But behind the scenes, PostgreSQL stores the data in smaller, physical “child” tables, or **partitions**. Each partition has a rule. For example, you could have a separate partition (a separate drawer) for each month's orders.

When you ask for January 2022's orders, PostgreSQL is smart enough to know it only needs to look in the “Jan 2022” drawer. It completely ignores all the other partitions. This is called **partition pruning**, and it's the source of partitioning's magic. The performance gains can be astronomical.

### Our Scenario → The PyShop orders Table

Let's model our problematic table. Before any partitioning, it probably looks something like this in SQLAlchemy:

```python
# models.py
import enum
from sqlalchemy import (
    create_engine,
    Column,
    Integer,
    String,
    DateTime,
    Enum,
    Numeric,
)
from sqlalchemy.orm import declarative_base
import datetime

Base = declarative_base()

class OrderStatus(enum.Enum):
    PENDING = "pending"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Order(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True)
    customer_id = Column(Integer, index=True, nullable=False)
    amount = Column(Numeric(10, 2), nullable=False)
    status = Column(Enum(OrderStatus), nullable=False, default=OrderStatus.PENDING)
    shipping_region = Column(String(2), nullable=False) # e.g., 'NA', 'EU', 'AS'
    created_at = Column(DateTime, default=datetime.datetime.utcnow, index=True)

    def __repr__(self):
        return f"<Order(id={self.id}, created_at={self.created_at.date()})>"
```

Simple enough. But with 500 million rows, even indexed queries on `created_at` start to feel sluggish. Let's fix this.

## The Flavors of Partitioning

PostgreSQL offers a few ways to slice up your data. The one you choose depends entirely on your data and how you query it.

### **1. Range Partitioning**

This is the most common type and perfect for our time-series data. You partition the data based on a continuous range of values, like dates or numbers. For our `orders` table, the `created_at` column is the perfect candidate. We'll create a separate partition for each month.

First, we need to tell SQLAlchemy and PostgreSQL that this table will be partitioned. We do this with a special `postgresql_partition_by` instruction.

```python
# models.py (The Partitioned Version)

class Order(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True)
    customer_id = Column(Integer, nullable=False) # No index needed on parent
    amount = Column(Numeric(10, 2), nullable=False)
    status = Column(Enum(OrderStatus), nullable=False, default=OrderStatus.PENDING)
    shipping_region = Column(String(2), nullable=False)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

    __table_args__ = {
        "postgresql_partition_by": "RANGE (created_at)"
    }
```

**Key Change →** We've removed the `index=True` from the columns in the parent table definition and added the `__table_args__` dictionary. The `id` is still the primary key for the logical table, but indexes are more efficient when created on the individual partitions.

When we create this table, PostgreSQL creates the “parent” orders table. It's an empty shell; it can't hold data itself. Now, we need to create the actual partitions (the drawers). You typically do this with raw SQL.

Let's create partitions for the first two months of 2024:

```sql
-- Partition for January 2024
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Partition for February 2024
CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- It's also a good practice to create indexes on each partition
CREATE INDEX ON orders_2024_01 (customer_id);
CREATE INDEX ON orders_2024_02 (customer_id);
```

Now, when you insert an order with a `created_at` of `2024-01-15`, it automatically goes into the `orders_2024_01` table. If you run a query like:

```python
# This query is now lightning fast!
session.query(Order).filter(
    Order.created_at >= '2024-01-01',
    Order.created_at < '2024-02-01'
).all()
```

Postgres's query planner looks at the `WHERE` clause and says, “Aha! I only need to scan the `orders_2024_01` partition.” All other partitions are ignored. Sweet, sweet relief.

### **2. List Partitioning**

List partitioning is used when you want to partition based on a specific, discrete list of values. Think of things like country codes, status types, or product categories. In our PyShop scenario, let's say our logistics team constantly runs queries based on the `shipping_region`. We could partition by that.

```sql
-- This is a hypothetical example
-- We'll stick with Range for our main 'orders' table.
CREATE TABLE orders_by_region (
    -- same columns as 'orders' ...
) PARTITION BY LIST (shipping_region);

CREATE TABLE orders_na PARTITION OF orders_by_region
    FOR VALUES IN ('NA'); -- North America

CREATE TABLE orders_eu PARTITION OF orders_by_region
    FOR VALUES IN ('EU'); -- Europe

CREATE TABLE orders_apac PARTITION OF orders_by_region
    FOR VALUES IN ('AS', 'OC'); -- Asia & Oceania

-- A default partition is a great safety net!
CREATE TABLE orders_other_regions PARTITION OF orders_by_region DEFAULT;
```

Now, a query for `WHERE shipping_region = 'EU'` will only touch the `orders_eu` partition.

### **3. Hash Partitioning**

Hash partitioning is the go-to when you don't have an obvious `Range` or `List` key, but you want to distribute data evenly across a fixed number of partitions. PostgreSQL takes the partition key, hashes it, and uses the result to decide which partition gets the row. This is useful for balancing write load. For example, if we wanted to partition our customers table by `customer_id` to prevent a single partition from getting all the write traffic for new customers.

```sql
CREATE TABLE customers_hashed (
    id int not null,
    name text,
    ...
) PARTITION BY HASH (id);

CREATE TABLE customers_h_0 PARTITION OF customers_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE customers_h_1 PARTITION OF customers_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE customers_h_2 PARTITION OF customers_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE customers_h_3 PARTITION OF customers_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

Data will be spread evenly across the four partitions. The downside is that partition pruning is less intuitive. It only works if your `WHERE` clause has an equality check (e.g., `WHERE id = 123`). Range queries won't benefit.

## Choosing the Right Partition Key

Your choice of partition key is the most important decision you will make. A bad key can lead to zero performance gain or even make things worse.

1. **Query, Query, Query:** Look at your most frequent and most performance-critical queries. The columns in your `WHERE` clauses are your primary candidates. For our reporting, it was `created_at`.
2. **Maintenance:** Think about data lifecycle. A date-based key like `created_at` is fantastic because it makes archiving or deleting old data trivial. Instead of a slow `DELETE FROM orders WHERE created_at < '2022-01-01'`, you can just run `DROP TABLE orders_2021_12;`. It's instantaneous.
3. **Avoid Unbalanced Partitions:** If you partition by status, your `'delivered'` partition might have 95% of the data, while `'cancelled'` has 1%. This is an unbalanced or skewed partition scheme and largely defeats the purpose.

## Level Up → Composite Partitioning

What if you need both? What if the PyShop logistics team wants fast queries for orders from a specific month *and* a specific region? You can partition a partition! This is called Composite Partitioning. Let's partition our `orders` table by `RANGE (created_at)` and then sub-partition each month by `LIST (shipping_region)`.

```sql
-- Step 1: Create the parent table with a composite key
CREATE TABLE orders (
    -- ... columns ...
) PARTITION BY RANGE (created_at);

-- Step 2: Create a monthly partition, but define its OWN partition scheme
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01')
    PARTITION BY LIST (shipping_region);

-- Step 3: Create the sub-partitions for that month
CREATE TABLE orders_2024_01_na PARTITION OF orders_2024_01 FOR VALUES IN ('NA');
CREATE TABLE orders_2024_01_eu PARTITION OF orders_2024_01 FOR VALUES IN ('EU');
CREATE TABLE orders_2024_01_other PARTITION OF orders_2024_01 DEFAULT;
```

Now, a query like this is absurdly efficient:

```sql
SELECT * 
FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'
  AND shipping_region = 'NA';
```

PostgreSQL will navigate directly to the tiny `orders_2024_01_na` table, ignoring everything else. This is peak database optimization.

## Don't Do It By Hand! Automating Partition Creation with Python

So, we're partitioning by month. What happens on August 1st, 2025? An unsuspecting `INSERT` will fail because the `orders_2025_08` partition doesn't exist yet. Whoops.

Manually creating a new partition every month is a recipe for a 3 AM wakeup call. This is a job for automation! Here’s a simple Python script you could run via a cron job or a Celery task at the end of each month.

```python
# maintenance/create_partitions.py
import psycopg2
from psycopg2 import sql
from datetime import datetime, timedelta
from dateutil.relativedelta import relativedelta

# --- Your DB credentials ---
DB_SETTINGS = {
    "dbname": "pyshop_db",
    "user": "pyshop_user",
    "password": "your_secret_password",
    "host": "localhost",
    "port": "5432"
}

def create_monthly_order_partition(target_date):
    """Creates a new partition for the orders table for the month of target_date."""

    conn = None
    try:
        conn = psycopg2.connect(**DB_SETTINGS)
        cursor = conn.cursor()

        # Calculate start and end of the month for the partition range
        start_of_month = target_date.replace(day=1, hour=0, minute=0, second=0, microsecond=0)
        end_of_month = start_of_month + relativedelta(months=1)

        partition_name = f"orders_{start_of_month.strftime('%Y_%m')}"
        
        # Using psycopg2.sql to safely format identifiers and literals
        create_partition_sql = sql.SQL(
            """
            CREATE TABLE IF NOT EXISTS {partition} PARTITION OF orders
            FOR VALUES FROM ({start_date}) TO ({end_date});
            """
        ).format(
            partition=sql.Identifier(partition_name),
            start_date=sql.Literal(start_of_month.strftime('%Y-%m-%d')),
            end_date=sql.Literal(end_of_month.strftime('%Y-%m-%d'))
        )

        print(f"Executing: {create_partition_sql.as_string(cursor)}")
        cursor.execute(create_partition_sql)
        
        # Also create indexes for the new partition!
        create_index_sql = sql.SQL(
            "CREATE INDEX IF NOT EXISTS {index_name} ON {partition} (customer_id);"
        ).format(
            index_name=sql.Identifier(f"{partition_name}_customer_id_idx"),
            partition=sql.Identifier(partition_name)
        )

        print(f"Executing: {create_index_sql.as_string(cursor)}")
        cursor.execute(create_index_sql)
        
        conn.commit()
        print(f"Successfully created partition '{partition_name}'.")

    except psycopg2.Error as e:
        print(f"Database error: {e}")
        if conn:
            conn.rollback()
    finally:
        if conn:
            conn.close()

if __name__ == "__main__":
    # We want to ensure the partition for NEXT month always exists.
    # Run this script, e.g., on the 20th of every month.
    today = datetime.utcnow()
    next_month = today + relativedelta(months=1)
    
    print("--- Creating partition for current month (if not exists) ---")
    create_monthly_order_partition(today)
    
    print("\n--- Creating partition for next month (proactively) ---")
    create_monthly_order_partition(next_month)
```

Run this script with cron on the 25th of every month, and you'll never have to worry about a missing partition again.

## **Hold My Beer! Partitioning Pitfalls**

So, you're feeling pretty good. You've partitioned your orders table, your reports are flying, and you're the hero of the engineering team (Wait, OK let’s face it nobody even notices you 😄). It's easy to think you've found the ultimate database cheat code. This is the moment where a seasoned developer might say, “Hold my beer”, and try to partition everything. But like any powerful tool, partitioning can cause spectacular face-plants if you're not careful. Let's talk about the traps.

### **1. The Primary Key Puzzle**

This is the first, and most common, head-scratcher. You have an `id` column, and you want it to be a unique primary key. Simple, right? Not so fast. PostgreSQL has a dirty little secret: for a primary key or a unique constraint to be valid on a partitioned table, the partition key must be part of that constraint. So, this classic SQLAlchemy model will **FAIL** when you try to apply it to a table partitioned by `created_at`:

```python
# THIS WILL THROW AN ERROR!
class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True) # <-- The problem child
    created_at = Column(DateTime)
    # ... other columns
    __table_args__ = {
        "postgresql_partition_by": "RANGE (created_at)"
    }
```

Postgres will complain with an error like ERROR: unique constraint on partitioned table “orders” must include all partitioning columns.

**Why?** Because Postgres only checks for uniqueness within a single partition. It has no efficient way to guarantee that an id of `123` in the `orders_2024_01` partition doesn't also exist in the `orders_2024_02` partition without scanning all of them, which defeats the purpose.

**The Fix:** You have to create a composite primary key.

```python
# The Correct Way
class Order(Base):
    __tablename__ = "orders"
    # The primary key now includes the partition key
    id = Column(Integer, primary_key=True)
    created_at = Column(DateTime, primary_key=True) # <-- Now part of the PK
    # ...
```

This ensures uniqueness across the entire logical table. The downside is that your primary key is now a bit more complex. Most ORMs handle this gracefully, but it's a fundamental shift you need to be aware of.

### **2. Forgetting the Magic Word → The `WHERE` Clause**

Partitioning only works if your queries include the partition key in the `WHERE` clause. If you write a query that doesn't filter by the partition key, the query planner just shrugs and says, “Well, I guess I have to check every single drawer in the filing cabinet”.

```python
# This query gets NO benefit from partitioning by created_at.
# It will scan every single monthly partition.
session.query(Order).filter(Order.customer_id == 42).all()
```

This can actually be slower than querying a non-partitioned table because of the added overhead of checking multiple partitions. You've got to train yourself: Always try to filter by the partition key!

### **3. The “It's Not Magic, It's Maintenance” Clause**

That nifty Python script we wrote to create new partitions? It's now a critical piece of your infrastructure.

- What if the cron job fails?
- What if the server it runs on goes down?
- What if a Daylight Saving Time bug messes up your date calculations?

If your script fails to create the `orders_2024_04` partition, all order insertions after March 31st at midnight will start failing. This is a guaranteed an alert at nighttime. You need to monitor this automation just as you would any other critical service.

### **4. Too Many Partitions**

If partitioning by month is good, partitioning by day must be better, right? And by hour, even better! Whoa there, slow down. While having more granular partitions can be beneficial for some use cases, having thousands of partitions creates its own overhead. The query planner takes longer to figure out which partitions to scan, and simple `SELECT` statements can become bloated with planning time. For most scenarios, like our PyShop, monthly or quarterly partitions are a sweet spot. Don't overdo it.

### **5. Migrating the Monolith**

This guide assumed you were setting up a new table. But what if you need to partition an existing orders table with a billion rows? You can't just run `ALTER TABLE` and go for coffee. The command would lock the table for hours or days, effectively taking your site down. Migrating a live, massive table to a partitioned structure is a delicate, multi-step dance that usually involves:

1. Creating a new, partitioned table (`orders_new`).
2. Setting up triggers to copy all new `INSERT`s/`UPDATE`s/`DELETE`s from the old table to the new one.
3. Backfilling the data from the old table to the new one in small, manageable chunks.
4. Once they are in sync, in a brief maintenance window, you swap the tables.

It's a project in itself. Tools like `pg_partman` can help, but it's a significant undertaking.

In short, partitioning is a professional-grade tool. It offers incredible performance gains, but it demands respect, planning, and a bit of operational diligence. Use it wisely, and it will serve you well.

## Final Thoughts

Partitioning isn't a silver bullet for all performance problems, but for large, growing tables with a clear access pattern, it's a game-changer. By moving from a single monolithic table to a well-structured set of partitions, you can dramatically improve query performance, simplify data maintenance, and keep your application humming along smoothly.

So next time you see a table growing out of control, don't panic. Just think of it as a messy drawer waiting for a proper filing cabinet. Go forth and partition!