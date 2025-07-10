# A Pythonic Approach to Scaling PostgreSQL From Monolith to Shards

Check this post on my blog [here](https://hevalhazalkurt.com/blog/a-pythonic-approach-to-scaling-postgresql-from-monolith-to-shards/).

<br>


Here is the situation. Your Python app, powered by the glorious duo of FastAPI/Django and PostgreSQL, is a hit. Users are flocking in, data is piling up, and your`SELECT`queries, which used to be cheetah-fast, are now moving with the grace of a sleepy sloth. You've added all the indexes you can think of, you've optimized your queries with`EXPLAIN ANALYZE`until your eyes bled, and you even gave your database server a pep talk and a RAM upgrade. But the slowness persists.

Houston, we have a scaling problem.

This is the moment when a seasoned developer starts whispering the forbidden word:**Sharding**. It sounds cool, complex, and a little bit terrifying. But what is it, really? And is it the silver bullet for your performance woes, or a bazooka you're about to aim at your own foot?

As a Python developer, you don't just write code; you build systems. Understanding how to scale your database is a crucial part of that. So, grab your favorite beverage, and let's demystify PostgreSQL sharding from a practical, Python-and-SQLAlchemy point of view.

## First Things First → What in the World is Sharding?

Let's start with a simple analogy. Imagine your app's data is a single, gigantic phone book for the entire world. Finding someone named “John Smith” would be a nightmare. Sharding is like splitting that giant book into smaller, country-specific phone books. Looking for “John Smith” in the “USA” phone book is suddenlywayfaster.

In technical terms:

> Sharding is a database architecture pattern where you horizontally partition your data across multiple, independent database servers.
> 

Each of these servers is called a**shard**. Each shard holds a unique subset of the total data. To your application, they mightlogicallylook like one big database, butphysically, they are separate entities.

This is different fromVertical Scaling, which is just buying a bigger, more powerful server like printing the phone book on bigger paper with a stronger binding. Sharding is**Horizontal Scaling** which is adding more servers to distribute the load like getting more phone books.

## The Million-Dollar Question →ShouldI Shard?

Before you go off and rewrite your entire data access layer, let's be real. Sharding is a big deal. It adds atonof complexity to your application and infrastructure. It's a solution for “we have too much data and traffic” problems, not “my query is poorly written” problems.

You should consider sharding when:

1. **Sheer Data Volume →**Your database is growing into multiple terabytes. A single node struggles with backups, maintenance, and even fitting on a single machine's disk.
2. **Write Throughput is Maxed Out →**You have so many incoming writes like new users, logs, events that a single PostgreSQL instance can't keep up, even with the beefiest hardware. Read replicas don't help with write load!
3. **Read Throughput isStillan Issue →** You've already implemented read replicas, but even they are overwhelmed by the number of connections or the size of the working set of data that needs to be in memory.
4. **Geographical Needs →**You have a global user base and need to store data in specific regions for legal reasons like GDPR or to reduce latency for users in different parts of the world. A shard in the EU for European users, a shard in the US for American users, etc.

When NOT to shard (yet):

- You haven't exhausted vertical scaling.
- You haven't properly indexed your tables.
- You haven't set up read replicas to offload read queries.
- Your bottlenecks are in your application code, not the database.
- **Most importantly →** You don't have a clear**Shard Key**.

## The Heart of the Matter → The Shard Key

This is the most critical decision you will make. TheShard Keyis the piece of data (a column in your table) that determines which shard a row of data will live on. It's the “country” in our phone book analogy.

Choosing a good shard key is an art. A good key:

- **Evenly distributes the data →**Prevents one shard from becoming a hotspot (getting all the traffic) while others sit idle.
- **Is present in most of your queries →**If you need to fetch data, you should ideally know the shard key. Otherwise, how would you know which phone book to look in? You'd have to searchallof them, which defeats the purpose!

For a typical Python-backed SaaS application, a fantastic shard key is often the`tenant_id`,`company_id`, or`user_id`. Why? Because most of your queries are probably already scoped to a specific tenant or user. 

- “Get all invoices for`company_id=123`”.
- “Get the profile for`user_id=456`”.

This is perfect! The data for one company lives together on one shard.

## Let's Get Our Hands Dirty: Application-Level Sharding with Python & SQLAlchemy

Okay, theory is nice, but we're developers. We want to see some code. Let's build a highly simplified sharding implementation at the application level.

**Our Use Case →** A multi-tenant SaaS application. Each tenant (`tenant_id`) gets their own little universe of data. This is a natural fit for sharding. We'll use a simple Hash-based sharding strategy.

**The Setup:**

1. Imagine we have two separate PostgreSQL databases running.
    - `postgresql://user:pass@host1:5432/my_app_shard_0`
    - `postgresql://user:pass@host2:5432/my_app_shard_1`
2. We need to create our tables onbothdatabases. This is a part of the operational complexity.

First, let's define our SQLAlchemy model. Nothing special here.

```python
# models.py
from sqlalchemy import Column, Integer, String, BigInteger
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    
    id = Column(BigInteger, primary_key=True)
    tenant_id = Column(Integer, nullable=False, index=True) # Our beloved Shard Key!
    username = Column(String(50), nullable=False)
    email = Column(String(100), nullable=False)

    def __repr__(self):
        return f"<User(id={self.id}, tenant_id={self.tenant_id}, username='{self.username}')>"
```

Now for the magic. We need a way to manage our database connections and route queries to the correct shard. Let's build a simple session manager.

```python
# db_router.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from contextlib import contextmanager

# In a real app, this would come from your config file
DB_URIS = {
    0: "postgresql://user:pass@host1:5432/my_app_shard_0",
    1: "postgresql://user:pass@host2:5432/my_app_shard_1",
}

# Create engine instances for each shard
engines = {shard_id: create_engine(uri) for shard_id, uri in DB_URIS.items()}
NUM_SHARDS = len(DB_URIS)

# Our simple sharding logic
def get_shard_id(tenant_id: int) -> int:
    """
    This is the core sharding function.
    Given a shard key (tenant_id), it returns the shard ID.
    A simple hash-based approach.
    """
    return tenant_id % NUM_SHARDS

# A session factory that returns a session for the correct shard
@contextmanager
def get_session(tenant_id: int):
    """
    The main entry point for our application code.
    Provides a SQLAlchemy session connected to the correct shard.
    """
    shard_id = get_shard_id(tenant_id)
    engine = engines.get(shard_id)
    
    if engine is None:
        raise ValueError(f"Invalid shard_id: {shard_id} calculated for tenant_id: {tenant_id}")

    Session = sessionmaker(bind=engine)
    session = Session()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```

**How do we use this?**

Now, in your application logic you'd use this context manager.

```python
# main.py 
from db_router import get_session
from models import User

def create_new_user(tenant_id: int, username: str, email: str):
    print(f"Creating user for tenant {tenant_id}...")
    
    with get_session(tenant_id) as session:
        new_user = User(tenant_id=tenant_id, username=username, email=email)
        session.add(new_user)
        # The session will auto-commit or rollback on exit
        
    print(f"User {username} for tenant {tenant_id} created successfully!")

def get_user_by_name(tenant_id: int, username: str):
    print(f"Fetching user '{username}' from tenant {tenant_id}...")
    
    # We MUST have the tenant_id to know which shard to query!
    with get_session(tenant_id) as session:
        user = session.query(User).filter_by(tenant_id=tenant_id, username=username).first()
        print(f"Found: {user}")
        return user

# --- Let's see it in action ---
# This user will go to shard 123 % 2 = 1
create_new_user(tenant_id=123, username="alice", email="alice@example.com")

# This user will go to shard 456 % 2 = 0
create_new_user(tenant_id=456, username="bob", email="bob@example.com")

# To get Alice, we MUST provide her tenant_id
get_user_by_name(tenant_id=123, username="alice")

# Trying to find Alice in Bob's tenant will fail (as it should!)
get_user_by_name(tenant_id=456, username="alice")
```

See the catch?You always need the shard key to find your data.Queries that don't have the shard key (`SELECT * FROM users WHERE username = 'alice'`) are now nightmare fuel. You'd have to queryevery single shardand aggregate the results in your Python code. This is slow, complex, and generally avoided.

## The Easy Mode, Using Extensions like Citus

The DIY approach is great for learning, but for a serious, production-grade system, it has drawbacks:

- **Cross-shard transactions?**Forget about it.
- **Cross-shard JOINs?**You'll be doing them manually in Python and crying.
- **Rebalancing data?**A huge manual effort.
- **Schema migrations?**You have to run them on every shard.

This is where PostgreSQL extensions come to the rescue. The most famous one isCitus Data.

Citus is an open-source extension that transforms PostgreSQL into a distributed, sharded database. The beauty of Citus is that it abstracts away most of the sharding complexity from your application.

With Citus, your architecture looks like this:

1. You have a cluster of PostgreSQL nodes (acoordinatorand severalworkers).
2. Your Python application connectsonly to the coordinator node, just like a regular PostgreSQL database. Your`DATABASE_URL`points to one place.
3. You tell Citus which table you want to shard and what the shard key is (e.g.,`SELECT create_distributed_table('users', 'tenant_id');`).
4. That's... pretty much it.

Now, when you run`INSERT`or`SELECT ... WHERE tenant_id = 123`, the Citus coordinator automatically routes the query to the correct worker node (shard). Your application code, including your SQLAlchemy models and sessions, remains blissfully unaware of the sharding happening under the hood. It just sees one giant, powerful database. Citus can even handle many cross-shard queries and some types of JOINs for you!

## To Shard, or Not to Shard?

Sharding is a powerful technique for achieving massive scale, but it's not a free lunch. It's a fundamental change in your application's architecture.

As a developer, here's your takeaway:

1. **Don't Rush It:**Sharding is a last resort. First, optimize your queries, add indexes, scale vertically, and use read replicas.
2. **Choose Your Shard Key Wisely:**Your`tenant_id`or`user_id`is probably your best friend here. This choice will define your life for years to come.
3. **Understand the Trade-offs:**Application-level sharding (the DIY Python way) gives you full control but adds a ton of complexity to your code.
4. **Consider the Big Guns:**For serious production use, look into solutions like the Citus extension for PostgreSQL. They handle the messy parts so you can focus on writing your application logic.

Sharding is a journey, not a destination. It's a sign of success, your app is so popular you're literally breaking the database. Yay! By understanding the principles, you're not just a coder; you're an architect ready to build systems that can grow to any scale.

