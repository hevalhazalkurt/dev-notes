# Instance vs Class vs Static Methods in Python

Check this post on my blog [here](https://hevalhazalkurt.com/blog/instance-vs-class-vs-static-methods-in-python/).

<br>

If you've ever found yourself staring at `@classmethod` and `@staticmethod` wondering, “Wait, when do I use which again?”, you're not alone. I’ve been there too. These method types are simple on the surface but hide a lot of subtle power that can make your code cleaner, more maintainable, and easier to test. 

Let’s walk through them one step at a time starting with a shared base class to keep things practical.

## Setting the Stage → The Base Class

We'll use a base class called `UserAccount`. Imagine this is part of a backend for a SaaS application managing users.

```python
class UserAccount:
    platform = "MySaaSPlatform"

    def __init__(self, username, email):
        self.username = username
        self.email = email
```

So far, this is a classic class with an `__init__` method. Let’s explore how each method type behaves by adding features step by step.

## Instance Methods → The Default Workhorse

### What it is:

Instance methods are what you normally write. They access and modify the specific instance of the class. The first argument is always `self`.

### Why use it:

When your method needs to read or write instance-specific data like user email or username, use this.

Let’s add one:

```python
class UserAccount:
    platform = "MySaaSPlatform"

    def __init__(self, username, email):
        self.username = username
        self.email = email
        
    def get_profile(self):
        return {
            "username": self.username,
            "email": self.email,
            "platform": self.platform
        }
```

Now we can use it like below.

```python
user = UserAccount("alice", "alice@example.com")
print(user.get_profile())

# {'username': 'alice', 'email': 'alice@example.com', 'platform': 'MySaaSPlatform'}
```

So, what's happening here?

- `self.username` and `self.email` are tied to that specific user
- `self.platform` is accessed from the class-level but available to all instances

Imagine you’re building a REST API. This could map directly to your `GET /user/{id}` route.

## Class Methods → Thinking in Blueprints

### What it is:

Class methods get the class as the first argument, usually named `cls`. That means they can create or manipulate the class itself, not just instances.

### Why use it:

- Alternate constructors like from a string, a dict, a config file etc.
- Factory patterns
- Managing class-level behavior or registration

Now let’s add one to our class.

```python
class UserAccount:
    platform = "MySaaSPlatform"

    def __init__(self, username, email):
        self.username = username
        self.email = email
        
    def get_profile(self):
        return {
            "username": self.username,
            "email": self.email,
            "platform": self.platform
        }
        
    @classmethod
    def from_dict(cls, data):
        return cls(data["username"], data["email"])
```

And we can use it like below.

```python
user_data = {"username": "bob", "email": "bob@example.com"}
user = UserAccount.from_dict(user_data)
```

What’s happening here?

- `cls` here refers to `UserAccount`, not an object.
- `cls(...)` creates a new instance. It’s useful if the class name ever changes or is inherited.

Let’s say your API receives JSON payloads. This is a great way to hydrate an object from them.

### Let’s test subclassing

```python
class AdminAccount(UserAccount):
    def __init__(self, username, email, admin_level=1):
        super().__init__(username, email)
        self.admin_level = admin_level
```

```python
admin_data = {"username": "root", "email": "root@example.com"}
admin = AdminAccount.from_dict(admin_data)
print(type(admin))  

# <class '__main__.AdminAccount'>
```

Because we used `cls`, the factory works with subclasses too. That’s powerful.

## Static Methods → Utility Without Context

### What it is:

Static methods don’t get `self` or `cls`. They’re just plain functions living inside a class.

### Why use it:

- Utility functions that logically belong to the class
- When you want organization without inheritance or context
- Formatting, parsing, validating, etc.

Now let’s add this one too to out codebase

```python
class UserAccount:
    platform = "MySaaSPlatform"

    def __init__(self, username, email):
        self.username = username
        self.email = email
        
    def get_profile(self):
        return {
            "username": self.username,
            "email": self.email,
            "platform": self.platform
        }
        
    @classmethod
    def from_dict(cls, data):
        return cls(data["username"], data["email"])
        
    @staticmethod
    def is_valid_email(email):
        return "@" in email and "." in email
```

And in practice we use it like below.

```python
print(UserAccount.is_valid_email("test@company.com"))  # True
```

What’s happening?

- No access to `self` or `cls`
- Think of it as a “related tool” rather than behavior

In a real-life example, we can validate data before creating a user which is so useful:

```python
data = {"username": "jane", "email": "janeexample.com"}

if UserAccount.is_valid_email(data["email"]):
    user = UserAccount.from_dict(data)
else:
    print("Invalid email format")
```

## Bringing It All Together

Imagine you're building a user registration flow in your backend.

```python
class UserAccount:
    platform = "MySaaSPlatform"

    def __init__(self, username, email):
        self.username = username
        self.email = email

    def get_profile(self):
        return {
            "username": self.username,
            "email": self.email,
            "platform": self.platform
        }

    @classmethod
    def from_request_payload(cls, payload: dict):
        if not cls.is_valid_email(payload["email"]):
            raise ValueError("Invalid email address.")
        return cls(payload["username"], payload["email"])

    @staticmethod
    def is_valid_email(email):
        return "@" in email and "." in email
```

You can use this class in your creating new user flow like that:

```python
def register_user(payload):
    try:
        user = UserAccount.from_request_payload(payload)
        return user.get_profile()
    except ValueError as e:
        return {"error": str(e)}
```

You’ve just:

- Validated email using a `staticmethod`
- Created a user with a `classmethod`
- Returned user info with an `instance method`

Everything is where it belongs, and the code reads like a story.

## Common Mistakes & Gotchas

| **Mistake** | **Why It's a Problem** |
| --- | --- |
| Using `@staticmethod` when you actually need access to the class or instance | Leads to rigid, unextendable code |
| Using instance method (`self`) for something that doesn’t use instance state | Misleads readers, makes testing harder |
| Forgetting that `classmethod` respects inheritance | Can lead to unexpected object types in factories |

## Summary Table

| **Type** | **First Arg** | **Accesses** | **Use Cases** |
| --- | --- | --- | --- |
| Instance (`def`) | `self` | instance + class attrs | Core behavior tied to object |
| Class (`@classmethod`) | `cls` | class + static context | Alternative constructors, factories |
| Static (`@staticmethod`) | None | nothing automatically | Utilities, validators, formatters |

## Final Thoughts

In modern Python applications, especially backend systems, the way you split logic into instance, class, and static methods can drastically affect readability, testability, and maintainability.

Think of it like this:

- **Instance methods** answer: “What can this user do?”
- **Class methods** answer: “How do I create or manage users?”
- **Static methods** answer: “How do I validate or process data related to users?”

Master this trio, and you'll find yourself writing cleaner, more scalable code with no more confusion, no more hacks.