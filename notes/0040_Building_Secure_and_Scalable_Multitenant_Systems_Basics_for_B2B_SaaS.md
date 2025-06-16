# Building Secure and Scalable Multitenant Systems Basics for B2B SaaS

# Why Multitenancy Matters

Imagine you're building a SaaS CRM tool for businesses. Every customer expects their data to be private and isolated, even though you're serving them all from a single backend system. This need for isolation, without deploying a separate application for each customer, is what makes multitenancy essential.

Multitenancy allows your application to support multiple independent tenants, usually companies or organizations, using a shared set of infrastructure like application code, servers, databases, while keeping their data isolated. It's not just about saving money on infrastructure; it's about scalability, maintainability, and security. Choosing the right multitenancy strategy early can dramatically reduce complexity as your application grows.

As a backend developer, your job isn't just to implement endpoints. It's to architect systems that are safe, scalable, and tenant-aware.

# Core Concepts of Multitenancy

## What is a Tenant?

In the context of multitenancy, a tenant is a single customer or organization that uses your application. Each tenant expects:

- their own users, roles, and permissions
- their own business data like leads, invoices, reports
- their own configurations like branding, billing cycles, etc.

Your backend system should ensure that tenant data is always isolated, even though the codebase is shared.

## The Three Main Database Models

There are three primary strategies to implement multitenancy at the database level. Each comes with trade-offs.

### 1. Shared Database, Shared Schema

All tenants use the same database and the same tables. You distinguish tenant data by adding a `tenant_id` column to each table.

```sql
SELECT * FROM invoices WHERE tenant_id = 'acme_corp';
```

**Pros:**

- simple to implement
- efficient use of database resources
- easy to scale early on

**Cons:**

- all queries must include `tenant_id`, or you'll risk data leaks
- difficult to enforce tenant isolation with database-level permissions
- migrations affect all tenants at once

This model is good for early-stage products or internal tools where complexity needs to be kept low.

### 2. Shared Database, Separate Schemas

Here, all tenants share the same database, but each tenant has their own schema (a namespace of tables).

```sql
-- Example: schema tenant_acme
SELECT * FROM tenant_acme.invoices;
```

**Pros:**

- stronger isolation than shared schema
- you can manage permissions per schema
- tenants can have slightly customized table definitions

**Cons:**

- you'll need to manage many schemas as tenant count grows
- schema migrations are more complex (must be applied per schema)
- some ORMs like Django ORM don't natively support schema switching

This is a good compromise when you need more isolation but don't want the operational cost of multiple databases.

### 3. Separate Database per Tenant

Each tenant gets their own dedicated database. You provision a new database when a customer signs up.

```python
def get_engine(tenant_id):
    db_url = f"postgresql://user:pass@host/tenant_{tenant_id}"
    return create_engine(db_url)
```

**Pros:**

- best isolation and security like per-database backups
- tenants can scale independently
- schema changes can be rolled out incrementally

**Cons:**

- higher operational cost
- connection pool management becomes harder
- requires infrastructure automation like provisioning, migrations

This model is ideal for enterprise-grade SaaS platforms, especially when clients require guaranteed isolation.

## A Real-World Scenario: SaaS CRM Platform

Let’s say you’re building a CRM app for B2B clients. Each tenant needs to:

- add and manage leads
- invite internal users
- run custom reports

We'll walk through how each multitenancy model shapes your architecture using this scenario.

### Shared Schema Implementation

All tenant data lives in shared tables, distinguished by `tenant_id`:

```python
class BaseModel(Base):
    __abstract__ = True
    tenant_id = Column(String, nullable=False)

class Lead(BaseModel):
    __tablename__ = 'leads'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)
```

Every query must filter on `tenant_id`:

```python
def get_leads_for_tenant(session, tenant_id):
    return session.query(Lead).filter_by(tenant_id=tenant_id).all()
```

Mistakes are easy here: forget to filter by `tenant_id`, and you may leak data between tenants.

### Schema-per-Tenant Implementation

Each tenant gets a unique schema: `tenant_acme`, `tenant_foobar`, etc.

You dynamically switch the schema in SQLAlchemy by setting `search_path` on connection:

```python
@event.listens_for(engine, "connect")
def set_schema(dbapi_connection, connection_record):
    schema = get_schema_for_request()
    cursor = dbapi_connection.cursor()
    cursor.execute(f"SET search_path TO {schema}")
    cursor.close()
```

This way, you use the same models, but they resolve to tenant-specific schemas. There's no need for a `tenant_id` field anymore. 

### DB-per-Tenant Implementation

Each signup triggers a new database creation:

```python
subprocess.run(["createdb", f"tenant_acme"])
```

When handling a request:

```python
def get_session_for_tenant(tenant_id):
    db_url = f"postgresql://.../tenant_{tenant_id}"
    engine = create_engine(db_url)
    return sessionmaker(bind=engine)()
```

Every tenant has a fully isolated environment, but now you're managing hundreds or thousands of databases.

## Pros and Cons Comparison

Let’s look at the big picture. 

| **Model** | **Pros** | **Cons** |
| --- | --- | --- |
| Shared Schema | Easy to start, efficient | Risk of cross-tenant leaks, harder compliance |
| Schema-per-Tenant | Better isolation, easier RLS | Migration + tooling complexity |
| DB-per-Tenant | Strongest isolation | Highest ops + infra cost |

There is no “best” model, only trade-offs based on your product, team size, and customer expectations.

So there are some common pitfalls to watch out for. 

1. **Leaking data between tenants**
    - This is especially common in shared schema setups.
    - Always use middleware to inject `tenant_id` into every query.
2. **Global cache collisions**
    - Don’t store shared Redis keys like `"leads_count"`; use tenant-specific keys like `"acme:leads_count"`.
3. **Hard-to-maintain migrations**
    - With schema-per-tenant or DB-per-tenant, you must apply migrations many times.
    - Consider tools like `alembic` + scripts to iterate per schema.
4. **Exhausting connection pools**
    - DB-per-tenant models can blow up pool limits. Use `pgbouncer` or pool sharing strategies.


## Conclusion

Multitenancy is not just about data architecture, it's about trust. Your tenants trust that their data is isolated, secure, and performant, regardless of how many other customers you're hosting. Start simple if you're early, but design for the long term. Build abstractions that allow you to evolve from shared schema to schema-per-tenant or DB-per-tenant as needed. Most importantly, treat multitenancy as a first-class concern in your architecture not just a detail in your models.