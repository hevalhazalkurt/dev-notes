# One Menu, Two Baristas: Handling SQL & NoSQL with the Repository Pattern

Check this post on my blog [here](https://hevalhazalkurt.com/blog/one-menu-two-baristas-handling-sql-nosql-with-the-repository-pattern/).

<br>


After a little vacation break, I wanted to write about the Repository Pattern today. For the longest time, it was one of those things I was doing without really knowing. You know the feeling, right? It's a classic symptom of learning on the job: you can build the thing, make it work, and ship it, but when someone asks you to explain the theory behind it, you start waving your hands and saying something stupid. 

Anyway, as I was thinking about what kind of example I could build for this post, I glanced over at my desk. And there it was. My ever-present, ever-faithful companion: a mug of coffee. Mostly cold, half-finished, and forgotten mid-sip while untangling some gnarly bug. And just like that, I had my story. A tale of coffee, code, and the architectural pattern that can save your project from a world of pain.

So, grab your own (hopefully hot) cup of coffee, and let's talk about the Repository Pattern through the lens of a fictional startup: ”**Code & Coffee LLC”**

### **Our Core Needs**

1. **A Catalog of Coffee Beans:** This is our primary data. We need to store things like the bean's `name`, `origin`, `roast_level`, and `stock_quantity`. This data is structured, relational, and perfect for a good old SQL database like PostgreSQL (my fav).
2. **User Profiles & Subscriptions:** Standard stuff. User emails, hashed passwords, and subscription tiers. Also a great fit for our SQL database.
3. **The “Killer Feature” - User Tasting Notes:** This is where it gets tricky. We want users to be able to log detailed tasting notes for each coffee they try. This data is complex, semi-structured, and often nested. A user might log notes like: 
    
    ```json
    {
      "aroma": [
        "chocolate",
        "nutty"
      ],
      "acidity": {
        "level": "medium",
        "notes": "citrusy aftertaste"
      },
      "mouthfeel": "smooth"
    }
    ```
    
    For this kind of flexible, JSON-heavy data, a NoSQL document database like MongoDB would be a much better fit than trying to shoehorn it into relational tables.
    

So, our application needs to talk to two different databases simultaneously. Our service layer needs to be able to say, “Get me this user's profile from Postgres, and then fetch all their tasting notes from Mongo”.

This is where a naive approach falls apart. If our `UserService` is full of SQLAlchemy code and our `TastingNoteService` is full of PyMongo code, they become tightly coupled to their respective databases. Swapping one out, or even just testing them in isolation, becomes a nightmare.

This is the perfect problem for a repository pattern to solve. Our goal is to create a clean, consistent data access layer that allows our business logic to interact with both PostgreSQL and MongoDB without ever knowing the difference. It's about building a team of specialized “baristas” who know how to work with their specific machines, while our application only ever has to place an order from a single, unified menu.

Before diving in and see how we build it, let’s check the basics.

## **What is the Repository Pattern?**

In the simplest terms, the Repository Pattern is a layer of abstraction between your application's business logic and your data access logic. It pretends to be an in-memory collection of your objects. 

It’s like a skilled barista standing between you (the customer/service layer) and the complex, noisy espresso machines (the databases). You don't walk behind the counter and start pulling levers yourself. You just tell the barista, “I'd like a double-shot latte”. You've stated your need in simple terms. The barista, an expert in their craft, knows exactly which machine to use, how to grind the beans, steam the milk, and assemble your drink.

That’s what a repository does. It pretends to be a simple, in-memory collection of your objects. Your service layer can say, “Hey, Repository, give me all the coffee beans from Ethiopia”, or “Add this new Kenyan bean to the collection”. The service layer doesn't know, and more importantly, doesn't care, if the repository is getting that data by running a SQL query on a PostgreSQL table, querying a document in a MongoDB collection, or even reading a CSV file from an S3 bucket.

This separation is the key. It decouples your core application from your data source, giving you incredible flexibility and making your code way easier to test and maintain.

To really get it, let's break it down into its core components. Think of them as the building blocks for our coffee shop:

1. **The Entity (The “Thing” Itself):** This is the core domain object we're working with. In our case, it's a `CoffeeBean` or a `User`. In an ORM like SQLAlchemy, this is your model class. It represents the actual data structure.
2. **The Repository Interface (The “Menu”):** This is the contract. It's an abstract class that defines what you can do, but not how. Our menu will say you can `get_bean_by_name` or `add_bean`, but it won’t have any of the messy details. This is the single most important piece for achieving decoupling, because our application will be built to work with this menu, not a specific barista.
3. **The Concrete Repository (The “Barista”):** This is the worker class that actually implements the interface. You’ll have one of these for each data source. Our `SQLAlchemyCoffeeBeanRepository` will be our PostgreSQL expert, knowing all the right SQLAlchemy incantations. Our `MongoCoffeeBeanRepository` will be our NoSQL guru, fluent in PyMongo. Each is a specialist, but both follow the same menu.
4. **The Service Layer (The “Customer”):** This is your business logic. The `CoffeeService` or `UserService` is the customer who walks up to the counter. The crucial rule is that the customer only ever looks at the menu (the interface). They trust that whoever is behind the counter can fulfill the order. This is what allows us to swap out our SQL barista for our Mongo barista without the customer even noticing.

By assembling our application with these four pieces, we build a system where changing our database technology goes from being a terrifying, project-halting rewrite to a manageable task of simply hiring a new barista and putting them behind the counter.

Enough tech details, let’s start building!

## Chapter 1: The Repository Interface (aka Writing the Menu)

Before we hire our first barista, we need to write the menu. This menu tells everyone what they can order. In code, this menu is our interface, or in Python, an Abstract Base Class (ABC). It defines a contract that any repository we create must follow.

Let's define a menu for our coffee beans.

```python
# /repositories/abstract_repository.py
import abc
from typing import List, Optional
from ..domain.models import CoffeeBean 

class AbstractCoffeeBeanRepository(abc.ABC):
    """This is our menu—the contract all our baristas must follow"""

    @abc.abstractmethod
    def add(self, bean: CoffeeBean) -> None:
        """Adds a new coffee bean to our collection"""
        raise NotImplementedError

    @abc.abstractmethod
    def get_by_name(self, name: str) -> Optional[CoffeeBean]:
        """Finds a coffee bean by its unique name"""
        raise NotImplementedError

    @abc.abstractmethod
    def list_by_origin(self, origin: str) -> List[CoffeeBean]:
        """Lists all coffee beans from a specific origin"""
        raise NotImplementedError
```

This is beautiful because it contains zero implementation. It's just a set of rules. It’s the promise of what our data layer can do.

## Chapter 2: The SQLAlchemy Implementation (aka Hiring Our First Barista)

Okay, we have our menu. Now, let's hire our first barista, one who is an expert in PostgreSQL and SQLAlchemy. This is our concrete implementation.

First, our SQLAlchemy model:

```python
# /domain/models.py
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class CoffeeBean(Base):
    __tablename__ = "coffee_beans"
    id = Column(Integer, primary_key=True)
    name = Column(String, unique=True)
    origin = Column(String)
    # ... other fields
```

Now, let's create the repository that knows how to talk to this table.

```python
# /repositories/sql_repository.py
from typing import List, Optional
from sqlalchemy.orm import Session
from .abstract_repository import AbstractCoffeeBeanRepository
from ..domain.models import CoffeeBean

class SQLAlchemyCoffeeBeanRepository(AbstractCoffeeBeanRepository):
    """Our first barista, an expert in all things SQL"""
    def __init__(self, session: Session):
        # We inject the database session because this repo can't work without it
        self.db = session

    def add(self, bean: CoffeeBean) -> None:
        self.db.add(bean)

    def get_by_name(self, name: str) -> Optional[CoffeeBean]:
        return self.db.query(CoffeeBean).filter(CoffeeBean.name == name).first()

    def list_by_origin(self, origin: str) -> List[CoffeeBean]:
        return self.db.query(CoffeeBean).filter(CoffeeBean.origin == origin).all()
```

This class follows the contract defined by `AbstractCoffeeBeanRepository` and contains all the nitty-gritty SQLAlchemy code. Our business logic never has to see this.

## Chapter 3: The Service Layer (aka The Customer Who Trusts the Menu)

Now, let's look at our `CoffeeService`, the part of our application that contains the business logic. The most important thing here is that it depends on the interface (the menu), not the concrete SQL implementation.

```python
# /services/coffee_service.py
from ..repositories.abstract_repository import AbstractCoffeeBeanRepository
from ..domain.models import CoffeeBean

class CoffeeService:
    def __init__(self, repo: AbstractCoffeeBeanRepository):
        # We depend on the menu, not a specific barista
        self.repo = repo

    def check_if_bean_exists(self, name: str) -> bool:
        """A simple business logic check"""
        bean = self.repo.get_by_name(name)
        return bean is not None

    def get_recommendations_for_origin(self, origin: str) -> list:
        """Finds all beans from an origin"""
        beans = self.repo.list_by_origin(origin)
        # Maybe do some complex logic here to create recommendations...
        return [bean.name for bean in beans]
```

Look at that! The `CoffeeService` has no idea SQLAlchemy exists. It’s just talking to its friendly barista through the methods defined on the menu.

## Chapter 4: Hiring a New, NoSQL-Savvy Barista

Our startup is a hit! We now need to implement that flexible "tasting notes" feature, and we’ve decided to move our `coffee_beans` data to MongoDB. Do we panic? No! We just hire a new barista. Let's create a new repository that implements the same interface but uses `pymongo` to talk to MongoDB.

```python
# /repositories/mongo_repository.py
from typing import List, Optional
from pymongo.collection import Collection
from .abstract_repository import AbstractCoffeeBeanRepository
from ..domain.models import CoffeeBean 

class MongoCoffeeBeanRepository(AbstractCoffeeBeanRepository):
    """Our trendy new NoSQL barista"""
    def __init__(self, collection: Collection):
        self.collection = collection

    def add(self, bean: CoffeeBean) -> None:
        # Pydantic model to dict, then insert
        self.collection.insert_one(bean.model_dump())

    def get_by_name(self, name: str) -> Optional[CoffeeBean]:
        doc = self.collection.find_one({"name": name})
        return CoffeeBean(**doc) if doc else None

    def list_by_origin(self, origin: str) -> List[CoffeeBean]:
        docs = self.collection.find({"origin": origin})
        return [CoffeeBean(**doc) for doc in docs]
```

The implementation code is completely different, but it respects the exact same menu. This is the magic of the pattern.

## Chapter 5: The One-Line Change That Saves the Day

So, how do we switch our entire application from PostgreSQL to MongoDB? In a modern framework like FastAPI that uses Dependency Injection, it's literally a one-line change in the place where you wire up your dependencies.

**Old Wiring (using SQL):**

```python
# /main.py 
from .repositories.sql_repository import SQLAlchemyCoffeeBeanRepository
from .repositories.abstract_repository import AbstractCoffeeBeanRepository
from .database import get_db_session # a function that yields a SQLAlchemy session

def get_coffee_repository(session: Session = Depends(get_db_session)) -> AbstractCoffeeBeanRepository:
    return SQLAlchemyCoffeeBeanRepository(session) # we provide the SQL barista
```

**New Wiring (using Mongo):**

```python
# /main.py 
from .repositories.mongo_repository import MongoCoffeeBeanRepository 
from .repositories.abstract_repository import AbstractCoffeeBeanRepository
from .database import get_mongo_collection # a function that yields a Mongo collection

def get_coffee_repository(collection: Collection = Depends(get_mongo_collection)) -> AbstractCoffeeBeanRepository:
    return MongoCoffeeBeanRepository(collection) # change this ONE LINE
```

That's it. We changed one line. Our `CoffeeService` remains 100% untouched. It asked for a barista that follows the menu, and our dependency injection system gave it the new MongoDB expert. It has no idea anything changed, yet the entire data layer was swapped out from under it.

## **Conclusion**

The Repository Pattern isn't just a technical curiosity; it's a safety net. It's the architectural choice that lets you say “yes” when your co-founder comes to you with that crazy-but-brilliant idea to switch data technologies. It separates what your application does from how the data is stored, giving you a system that is robust, testable, and ready for the future.

So next time you start a project, think about hiring a good barista. But don’t forget to sure you “really” need it. If you're building a simple script or a throwaway prototype where you know the database will never, ever change, it might be overkill. Patterns are tools, not commandments. Use them when they solve a real problem.