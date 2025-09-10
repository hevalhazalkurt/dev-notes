# Webhooks: “Don't Call Us, We'll Call You”

Check this post on my blog [here](https://hevalhazalkurt.com/blog/webhooks-dont-call-us-well-call-you/).

<br>


> “Is it done yet? How about now? Now?”
> 

I once worked on a system with a background job that processed user-uploaded files. The job's duration could vary wildly depending on the file size. Meanwhile, the front-end would poll our backend every few seconds to ask for a status update. When you consider that some of these jobs took close to an hour, you can imagine the sheer number of pointless requests hammering our backend. It felt like an endless, noisy crowd asking, “Are we there yet?”

If you've ever built (or suffered from) a system like that, you've experienced the pain of the “pull” model. You constantly poll an API, wasting resources, cluttering logs, and adding latency, all for a stream of “not yet” answers.

There’s a more elegant, more mature way to handle this, the “push” model. And its most common implementation in the web world is the humble, yet powerful, webhook.

In this post, we're not just going to define what a webhook is. We're going to build a practical scenario within a mock e-commerce backend, see exactly where it shines, and write some clean FastAPI code to bring it to life. Let's stop building noisy, impatient systems and start building ones that communicate like adults.

## Our Scenario → The Orderly E-commerce Platform

Let's build a mental model of our backend. We have two distinct microservices:

1. **`OrdersService`**: The star of the show. It handles creating orders, processing payments, and managing order status. It's written in Python using FastAPI.
2. **`NotificationService`**: A simple but crucial service. Its only job is to send notifications to customers like email, SMS, etc. It doesn't care why it's sending a message, only that it was told to.

**The Problem →** When an order is successfully paid for in the `OrdersService`, we need to tell the `NotificationService` to send a confirmation email to the customer.

How do we build this communication link? We could make the `OrdersService` call the `NotificationService` directly. But what if the `NotificationService` is down? Should the order payment fail? Absolutely not. Payment is critical; sending an email is secondary.

This is a perfect use case for a webhook. We'll decouple these services. The `OrdersService`'s only job is to process the order and then shout into the void, “Hey, anyone who cares, Order #123 was just paid for!” The `NotificationService` will be the one listening.

## The Core Idea → Pull vs. Push

Before deep dive to the codes let’s check the core idea to understand why webhooks are so effective, let's first consider the alternative in the context of our e-commerce platform. Our goal is simple: when the `OrdersService` confirms a payment, the `NotificationService` needs to send an email.

### **The Pull Model**

Without webhooks, the `NotificationService` would have to constantly ask the `OrdersService` for updates. This would look something like this:

1. The `OrdersService` would need to expose an endpoint, maybe `GET /api/orders/{id}/status`.
2. The `NotificationService` would have to keep a list of every single order it might need to send a notification for.
3. It would then have to repeatedly poll the `OrdersService` for each of these orders. The conversation would be:
    - `NotificationService` -> `GET /api/orders/123/status`
    - `OrdersService` <- `{"status": "pending_payment"}`
    - *(a few seconds later)*
    - `NotificationService` -> `GET /api/orders/123/status`
    - `OrdersService` <- `{"status": "pending_payment"}`
    - (again and again the same request-response)

This is a nightmare at scale. It generates a massive amount of useless network traffic and puts a constant, unnecessary load on the `OrdersService`, forcing it to answer the same question over and over. The `NotificationService` also becomes more complex, as it now needs to manage the state of this polling logic.

### **The Push Model**

Now, let's flip the script with the push model. This is where webhooks come in.

1. The `NotificationService` exposes a single endpoint, something like `POST /webhook/order-paid`. It then passively listens.
2. The `OrdersService`, upon successfully processing a payment for order #123, takes the initiative.
3. It immediately packages up the relevant details like the order ID, customer email, amount into a JSON payload.
4. It sends a single `HTTP POST` request directly to the `NotificationService`'s listening URL.

The conversation is now short, direct, and efficient:

- `OrdersService` -> `POST /webhook/order-paid` with order data.
- `NotificationService` <- `{"status": "event_received"}`

One action, one notification. Clean, immediate, and event-driven. The `OrdersService` doesn't have to answer endless questions, and the `NotificationService` doesn't have to ask.

This exact mechanism, one service sending an automated HTTP request to another in response to an event, is a webhook. Think of it as a “user-defined HTTP callback”. The user here is the `NotificationService`. It effectively tells the `OrdersService`, “*When an order is paid, call me back at this URL*”. This creates a powerful, decoupled subscription model that we're about to build.

## The Implementation

### Step 1 → Building the Listener, The Notification Service

First, let's create the endpoint in our `NotificationService` that will listen for these events. This is the webhook receiver or webhook listener. It's just a standard FastAPI endpoint that's ready to catch a `POST` request.

```python
# file: notification_service/main.py

from fastapi import FastAPI, Request, HTTPException
from pydantic import BaseModel, EmailStr
import logging

# configure basic logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Notification Service")

# this Pydantic model defines the contract of our webhook.
# the OrdersService MUST send data in this shape.
class OrderPaidEvent(BaseModel):
    order_id: int
    customer_email: EmailStr
    amount: float
    product_name: str

@app.post("/webhook/order-paid")
async def handle_order_paid_webhook(event: OrderPaidEvent, request: Request):
    """
    this is our webhook listener. it receives the 'order_paid' event
    and triggers the notification logic.
    """
    client_host = request.client.host
    logger.info(f"Received 'order_paid' event for order {event.order_id} from {client_host}")

    # -- real-world logic would be here --
    # 1. validate the request (is it coming from a trusted source?)
    # 2. add the notification task to a background queue (like Celery or ARQ)
    # 3. for now, we'll just simulate sending an email.

    print(f"Sending confirmation email to {event.customer_email} for order #{event.order_id}...")
    print(f"Product: {event.product_name}, Amount: ${event.amount:.2f}")

    # let the caller know we've successfully received and accepted the event.
    return {"status": "event_received"}
```

So what we’ve have done here?

**The Contract →** The `OrderPaidEvent` Pydantic model is our contract. It dictates the exact JSON structure the `OrdersService` must send. If it sends something different, FastAPI will automatically return a 422 Unprocessable Entity error.

**POST Endpoint →** There's no magic here. A webhook listener is just a normal API endpoint designed to be called by another machine, not a user.

**Acknowledge and Defer →** The listener's job is to receive the event, validate it, and then quickly hand off the actual work (sending the email in our case) to a background worker. It should respond with a 2xx status code as fast as possible to let the sender know the event was accepted. You never want the `OrdersService` waiting for an email to be sent.

### Step 2 → Firing the Webhook, The Orders Service

Now, let's modify the `OrdersService` to fire this webhook when an order is processed. In a real application, you'd store the webhook URLs in a database, allowing other services to subscribe to your events. For this example, we'll hardcode it.

```python
# file: orders_service/main.py

from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel, EmailStr
import httpx 
import os

app = FastAPI(title="Orders Service")

# get the webhook URL from an environment variable for better configuration
NOTIFICATION_WEBHOOK_URL = os.environ.get("NOTIFICATION_WEBHOOK_URL", "http://localhost:8001/webhook/order-paid")

class Order(BaseModel):
    product_name: str
    quantity: int
    price_per_item: float
    customer_email: EmailStr

# a simple async function to fire the webhook
async def fire_webhook(url: str, payload: dict):
    """
    sends a POST request to the specified webhook URL
    """
    async with httpx.AsyncClient() as client:
        try:
            print(f"Firing webhook to {url} with payload: {payload}")
            response = await client.post(url, json=payload, timeout=5.0)
            # raise an exception for 4xx/5xx responses
            response.raise_for_status()  
            print(f"Webhook delivered successfully. Status: {response.status_code}")
        except httpx.RequestError as e:
            print(f"Error firing webhook to {e.request.url!r}: {e}")

@app.post("/orders")
async def create_order(order: Order, background_tasks: BackgroundTasks):
    """
    simulates creating and paying for an order.
    """
    # 1. business logic -> save order to DB, process payment, etc
    order_id = 123  
    total_amount = order.quantity * order.price_per_item
    print(f"Order #{order_id} created for {order.customer_email}. Total: ${total_amount:.2f}")
    print("Payment processed successfully.")

    # 2. prepare the webhook payload based on the contract
    event_payload = {
        "order_id": order_id,
        "customer_email": order.customer_email,
        "amount": total_amount,
        "product_name": order.product_name,
    }

    # 3. fire the webhook in the background
    background_tasks.add_task(fire_webhook, NOTIFICATION_WEBHOOK_URL, event_payload)

    # 4. return a fast response to the user
    return {"message": "Order created successfully!", "order_id": order_id}
```

Let’s check what we have done on this part.

**`BackgroundTasks` →** We inject `BackgroundTasks` into our endpoint. When we call `background_tasks.add_task()`, FastAPI returns the HTTP response to the user immediately and then runs our `fire_webhook` function in the background. This is non-blocking and essential for a snappy user experience.

**Decoupling in action →** Notice that the `OrdersService` has zero knowledge of how notifications are sent. It doesn't know about email servers or SMS gateways. Its only responsibility is to send a well-structured JSON payload to a URL. We could replace the entire `NotificationService` tomorrow, and as long as the new one has the same webhook endpoint, the `OrdersService` wouldn't need to change a single line of code.

**Resilience through `httpx` →** Using a robust HTTP client like `httpx` is important. It handles timeouts and makes it easy to check for error responses, which is crucial for building a reliable system.

## The Tricky Bits → What Can Go Wrong?

This all looks great, but in the real world, webhooks come with their own set of challenges. As a senior developer, these are the things you need to be thinking about.

1. **Security → How do I know the webhook is legit?**
    
    Your public webhook endpoint is a target. Anyone could send a `POST` request to it and make you send fake emails.
    
    The solution is signature verification. The sender (`OrdersService`) should create a signature by hashing the request payload with a secret key (known only to both services). It sends this signature in a custom header like `X-Webhook-Signature`. The receiver (`NotificationService`) then performs the same hashing operation on the payload it receives and compares its result to the signature in the header. If they don't match, the request is a forgery and should be rejected with a `403 Forbidden`.
    
2. **Reliability → What if the receiver is down?**
    
    The `NotificationService` might be restarting or experiencing an outage when the `OrdersService` fires the webhook. The event is lost forever.
    
    The solution is retrying with exponential backoff. The sender must be prepared for failure. If a webhook call fails like a 5xx error or a timeout, it shouldn't just give up. It should retry the request. A good strategy is exponential backoff. So wait 1 second, then retry; if that fails, wait 2 seconds, then 4, then 8, and so on, up to a certain limit. This gives the receiving service time to recover. Libraries like Tenacity in Python can handle this automatically.
    
3. **Idempotency → What if I receive the same event twice?**
    
    Due to the retry logic above, it's possible for the `NotificationService` to receive the exact same `Order Paid` event more than once. If your logic is naive, you might send the customer two confirmation emails.
    
    The solution is idempotency key in that case. The sender should generate a unique ID for every event it sends like using a UUID and include it in the payload or a header as `X-Idempotency-Key`. The receiver should keep a record of the event IDs it has already processed. If a request comes in with an ID it's already seen, it can safely ignore it and just return a success response.
    

### So, When to Use Webhooks?

Webhooks are a fantastic tool, but they aren't a silver bullet. Use them when →

- You need asynchronous, real-time notifications.
- You want to decouple services to improve resilience and maintainability.
- The event producer should not be blocked waiting for the consumer's logic to complete.

Avoid them when →

- You need an immediate, synchronous response. If the `OrdersService` needed to know if the notification was successfully sent before proceeding, a direct API call (RPC) would be more appropriate.
- The caller needs to fetch data, not just be notified of an event. That's a job for REST or GraphQL.