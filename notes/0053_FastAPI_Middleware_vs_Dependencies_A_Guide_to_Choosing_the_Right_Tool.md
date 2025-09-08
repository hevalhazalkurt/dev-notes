# FastAPI Middleware vs. Dependencies, A Guide to Choosing the Right Tool

Check this post on my blog [here](https://hevalhazalkurt.com/blog/fastapi-middleware-vs-dependencies-a-guide-to-choosing-the-right-tool/).

<br>

As software engineers, we build upon layers of abstraction. We make countless decisions every day, and some of the most critical ones involve choosing the right tool for the job. No matter how much we don't admit it to ourselves, often we fall back on familiar patterns or the first solution that comes to mind. I recall an early project where I clumsily passed user authentication data through the `request.state` object from a middleware, leading to opaque code and difficult testing. It worked, but it was not the right tool. This experience taught me a valuable lesson: a deep understanding of your framework's architectural components is not an academic exercise, it is essential for building maintainable, scalable, and robust applications.

In the FastAPI ecosystem, two powerful mechanisms for handling cross-cutting concerns are middleware and global dependencies. On the surface, they seem to solve similar problems: running code on every request. However, their design, execution flow, and intended purposes are fundamentally different. Choosing incorrectly can lead to architectural dead-ends and code that fights against the framework's design principles. This article will provide a detailed comparison to help you make deliberate, informed decisions.

## Understanding the Core Concepts

Before we compare them, let's establish a clear definition of each component.

### What is Middleware?

In the context of a web framework like FastAPI, middleware is a function or class that wraps your entire application. It processes every incoming request before it reaches your application's routing logic and every outgoing response before it is sent to the client.

Think of it as a series of concentric layers. A request must pass through each layer of middleware to reach the core application, and the response must travel back out through those same layers in reverse order.

**Key Characteristics:**

- **Request/Response Cycle:** It has access to the raw Request object on the way in and the final Response object on the way out.
- **Routing Agnostic:** It executes before FastAPI has determined which specific path operation (endpoint) will handle the request. It has no knowledge of path parameters or the specific business logic that will run.
- **Broad Application:** It is ideal for concerns that are truly global and independent of your application's logic, such as logging, CORS headers, GZIP compression, or adding global headers to every response.

A basic middleware structure looks like this:

```python
from fastapi import Request
from starlette.responses import Response
from typing import Callable, Awaitable

async def my_middleware(request: Request, call_next: Callable[[Request], Awaitable[Response]]) -> Response:
    # Code to run before the request is processed by the application
    print("Processing request...")

    response = await call_next(request)

    # Code to run after the response has been generated
    print("Processing response...")
    return response
```

### What is a Dependency?

A dependency, in FastAPI, is a function that can be “injected” into your path operation functions. FastAPI's dependency injection (DI) system is one of its defining features. It allows you to write reusable pieces of code that provide data, perform validation, or establish connections (like a database session).

A **Global Dependency** is simply a dependency that is automatically applied to every path operation in your application or a specific router.

**Key Characteristics:**

- **Business Logic Context:** It runs after routing has been resolved but before your path operation's code is executed. It is designed to prepare the context and fulfill prerequisites for your endpoint.
- **Dependency Injection:** It can provide values (like a database session, the current user object) directly to your endpoint's parameters, ensuring type safety and clarity.
- **Specific Error Handling:** It can raise `HTTPException`, which FastAPI will gracefully handle, immediately stopping the request and returning a structured JSON error response. It does not have access to the final Response object if the endpoint succeeds.

A basic dependency structure looks like this:

```python
from fastapi import Depends, HTTPException, Header
from typing import Annotated

async def get_api_token(x_token: Annotated[str, Header()]) -> str:
    if x_token != "my-secret-token":
        raise HTTPException(status_code=401, detail="Invalid API Token")
    return x_token

# Usage in an endpoint
@app.get("/items/")
async def read_items(token: Annotated[str, Depends(get_api_token)]):
    return {"token": token}
```

## Base Scenario: Building a Multi-Tenant SaaS API

To illustrate the practical differences, let's establish a single, coherent scenario. We are building the backend for a multi-tenant SaaS application. Each client organization is a “tenant”. All API requests must be scoped to a specific tenant to ensure data isolation.

We have two specific technical requirements:

1. **Requirement A:** For monitoring and performance analysis, every API response must include a custom `X-Process-Time-MS` header indicating the total server-side processing time in milliseconds.
2. **Requirement B:** Every request to an authenticated endpoint must contain an `X-Tenant-ID` header. This ID must be validated against our database. If valid, we must establish a database session scoped to that tenant's data and make it available to the endpoint. If invalid, a `404 Not Found` error must be returned.

Let's analyze which tool is right for each requirement.

### Use Case 1 → Performance Monitoring Header

**The Task →** Implement the `X-Process-Time-MS` header.

This is a classic use case for middleware. We need to perform an action both before and after the core application logic runs; start a timer before the request is processed, and then calculate the elapsed time and add it to the response headers after the response has been generated.

**Why Middleware is the Right Choice…**

Because the crucial factor here is the need to modify the final `Response` object. Only middleware has access to the response after it has been fully formed by the path operation. A dependency's lifecycle ends before the response is created.

Let’s look at the implementation now.

```python
import time
from fastapi import FastAPI, Request
from starlette.responses import Response
from typing import Callable, Awaitable

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next: Callable[[Request], Awaitable[Response]]) -> Response:
    start_time_ns = time.time_ns()

    # Pass the request to the next step in the processing chain (routing, dependencies, endpoint)
    response = await call_next(request)

    # Once the response is generated, this code executes
    process_time_ms = (time.time_ns() - start_time_ns) / 1_000_000
    response.headers["X-Process-Time-MS"] = f"{process_time_ms:.2f}"

    return response

@app.get("/")
async def root():
    # Simulate some work
    await asyncio.sleep(0.05)
    return {"message": "Hello World"}
```

**What if we tried to use a dependency?**

It's simply not possible to implement this correctly with a dependency.

1. **No Response Access:** A dependency cannot see or modify the response. There is no response parameter to inject, and its execution is finished long before the response headers are finalized.
2. **Inaccurate Timing:** You could start a timer in a dependency, but you would have no clean place to stop it. The total processing time includes things like JSON serialization, which happens *after* the endpoint and its dependencies have finished. Any timing measured within a dependency would be incomplete.

### Use Case 2 → Tenant Scoping

**The Task →** Validate the `X-Tenant-ID` header and provide a tenant-scoped database session.

This requirement is a perfect fit for a global dependency. We need to perform validation and provide a resource (the database session) as a prerequisite for our business logic.

**Why a Global Dependency is the Right Choice:**

1. **Dependency Injection:** The primary goal is to provide a value to our path operations. The DI system is designed for precisely this. It makes the endpoint's signature explicit and type-safe like `async def get_items(db: Session = Depends(get_tenant_db))`.
2. **Clean Validation and Error Handling:** We can use FastAPI's automatic header parsing (`x_tenant_id: str = Header(...)`) and raise a standard `HTTPException` if validation fails. The framework handles the rest, returning a clean error response.
3. **Routing Awareness:** While not strictly needed here, dependencies run after routing, meaning they can access path parameters (e.g., `item_id: int`), which middleware cannot.

The implementation would look like that:

```python
class TenantNotFound(HTTPException):
    def __init__(self, tenant_id: str):
        super().__init__(status_code=404, detail=f"Tenant '{tenant_id}' not found.")

# This is our dependency function
async def get_tenant_db(x_tenant_id: Annotated[str, Header()]) -> Session:
    db = SessionLocal()
    tenant = db.query(Tenant).filter(Tenant.id == x_tenant_id).first()

    if not tenant:
        db.close() # Ensure session is closed even on failure
        raise TenantNotFound(tenant_id=x_tenant_id)

    # A real implementation would scope the session here, like using row-level security
    # For this example, we'll just yield it
    try:
        yield db
    finally:
        db.close()

# Apply it globally to an API router or the whole app
# router = APIRouter(dependencies=[Depends(get_tenant_db)])

@app.get("/items")
async def get_items(db: Annotated[Session, Depends(get_tenant_db)]):
    # 'db' is now a validated, tenant-scoped session.
    # We can safely use it to query items, knowing they belong to the correct tenant
    items = db.query(Item).all()
    return items
```

**What if we tried to use Middleware? (The Disadvantages)**

1. **Clunky Data Passing:** How would the middleware pass the db session to the endpoint? The common (but suboptimal) way is to attach it to `request.state.db`. This breaks type hinting and auto-completion in the endpoint signature. The explicit `Depends` is far superior for code clarity and static analysis.
2. **Awkward Error Handling:** To return a 404 error, the middleware would have to short-circuit the entire process by manually creating and returning a `JSONResponse`. This bypasses FastAPI's standard exception handling flow and is less elegant than a simple raise HTTPException.
3. **Mixing Concerns:** The middleware layer's responsibility is the raw HTTP lifecycle. Loading data models and interacting with the database is business logic preparation, which is the domain of dependencies. Using middleware for this violates the separation of concerns.

## Conclusion

Middleware and dependencies are not interchangeable competitors; they are complementary tools designed for different layers of your application.

- Choose **Middleware** when you need to operate on the raw request/response cycle, independent of your business logic. If your task involves modifying the final response, middleware is your only option.
- Choose **Global Dependencies** when you need to prepare the context for your business logic. If your task involves validation, authentication, or providing resources like a database session to your endpoints, the dependency injection system is the correct, idiomatic choice.