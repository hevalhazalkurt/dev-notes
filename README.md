# Dev-Notes
You can check [my personal blog](https://hevalhazalkurt.com/) for a better reading experience.

<br>


## Python 

| Article | Tags | 
|--|--|
| [Shallow vs Deep Copies in Python, What You Think You Know (but Might Not)](notes/0001_Shallow_vs_Deep_Copies_in_Python_What_You_Think_You_Know_but_Might_Not.md) | `copy`, `shallow copy`, `deep copy` |
| [Trash Talk: Understanding Python’s Garbage Collector](notes/0002_Trash_Talk_Understanding_Pythons_Garbage_Collector.md) | `gc module`, `memory`, `performance`|
| [The Power of `yield from` in Python Generators](notes/0003_The_Power_of_yield_from_in_Python_Generators.md) | `generators`, `yield from`, `fastapi`|
| [How Async/Await Evolved from Generator-Based Coroutines](notes/0004_How_Async_Await_Evolved_from_Generator_Based_Coroutines.md) | `generators`, `async` , `asyncio` |
| [How Order Changes Behavior in Chained Decorators](notes/0005_How_Order_Changes_Behavior_in_Chained_Decorators.md) | `generators`, `async`, `decorators`, `order`, `auth`, `logging`, `retry` |
| [Creating Declarative APIs with Class Decorators in Python](notes/0006_Creating_Declarative_APIs_with_Class_Decorators_in_Python.md) | `class`, `class decorators`, `declarative api`, `decorators`, `metaclass` |
| [Keys to Mastering Python Method Decorators](notes/0007_Keys_to_Mastering_Python_Method_Decorators.md) | `class`, `method decorators`,  `classmethod`, `staticmethod`, `property`, `factory methods`, `registry` |
| [The Danger of Overusing `is` Instead of `==` in Python](notes/0008_The_Danger_of_Overusing_is_Instead_of_==_in_Python.md) | `equality`, `comparison` |
| [Behind the Underscores EP01: Understanding Python’s Special Methods Conceptually](notes/0009_Behind_the_Underscores_EP01_Understanding_Pythons_Special_Methods_Conceptually.md) | `class`, `oop`, `special methods`, `dunder methods`|
| [Behind the Underscores EP02: Object Initialization and Construction Methods (`__new__`, `__init__`, `__del__`)](notes/0010_Behind_the_Underscores_EP02_Object_Initialization_and_Construction_Methods_new_init_del.md) | `class`, `oop`, `special methods`, `dunder methods`, `initialization`, `construction`|
| [Behind the Underscores EP03: String Representation Methods (`__str__`, `__repr__`, `__format__`)](notes/0011_Behind_the_Underscores_EP03_String_Representation_Methods_str_repr_format.md) | `class`, `oop`, `special methods`, `dunder methods`, `string`, `representation`|
| [Behind the Underscores EP04: Arithmetic Methods (`__add__`, `__sub__`, `__mul__`)](notes/0012_Behind_the_Underscores_EP04_Arithmetic_Methods_add_sub_mul.md) | `class`, `oop`, `special methods`, `dunder methods`, `math` |
| [Behind the Underscores EP05: Comparison Methods (`__eq__`, `__lt__`, `__gt__`)](notes/0013_Behind_the_Underscores_EP05_Comparison_Methods_eq_lt_gt.md) | `class`, `oop`, `special methods`, `dunder methods`, `comparison` |
| [Behind the Underscores EP06: Bitwise Methods (`__and__`, `__or__`, `__xor__`)](notes/0014_Behind_the_Underscores_EP06_Bitwise_Methods_and_or_xor.md) | `class`, `oop`, `special methods`, `dunder methods`, `bitwise`|
| [Behind the Underscores EP07: Container protocol (`__getitem__`, `__setitem__`, `__delitem__`)](notes/0015_Behind_the_Underscores_EP07_Container_protocol_getitem_setitem_delitem.md) | `class`, `oop`, `special methods`, `dunder methods`, `container`|
| [Behind the Underscores EP08: Length and iteration Methods (`__len__`, `__iter__`, `__next__`, `__contains__`)](notes/0016_Behind_the_Underscores_EP08_Length_and_iteration_Methods_len_iter_next_contains.md) | `class`, `oop`, `special methods`, `dunder methods`, `iteration`, `length`|
| [Behind the Underscores EP09: Attribute Access (`__getattr__`, `__getattribute__`, `__setattr__`, `__delattr__`)](notes/0017_Behind_the_Underscores_EP09_Attribute_Access_getattr_getattribute_setattr_delattr.md) | `class`, `oop`, `special methods`, `dunder methods`, `attribute access`|
| [Behind the Underscores EP10: Context Management (`__enter__`, `__exit__`)](notes/0018_Behind_the_Underscores_EP10_Context_Management_enter_exit.md) | `class`, `oop`, `special methods`, `dunder methods`, `context management`|
| [Behind the Underscores EP11: Callable Objects: `__call__`](notes/0019_Behind_the_Underscores_EP11_Callable_Objects_call.md) | `class`, `oop`, `special methods`, `dunder methods`, `callable`|
| [Behind the Underscores EP12: Descriptor Protocol (`__get__`, `__set__`, `__delete__`)](notes/0020_Behind_the_Underscores_EP12_Descriptor_Protocol_get_set_delete.md) | `class`, `oop`, `special methods`, `dunder methods`, `descriptor protocol`|
| [Behind the Underscores EP13: Metaprogramming Methods (`__class__`, `__bases__`, `__mro__`, `__instancecheck__`)](notes/0021_Behind_the_Underscores_EP13_Metaprogramming_Methods_class_bases_mro_instancecheck.md) | `class`, `oop`, `special methods`, `dunder methods`, `metaprogramming` |
| [Understanding Async Context Managers in Python](notes/0022_Understanding_Async_Context_Managers_in_Python.md) | `async`, `context manager`, `asyncio`, `connection pool`, `contextlib`, `fastapi`|
| [How the GIL Affects Real Python Workloads](notes/0023_How_the_GIL_Affects_Real_Python_Workloads.md) | `gil`, `performance`, `asyncio`, `threading`, `multiprocessing`, `fastapi`|
| [Slotted classes with `__slots__` in Python](notes/0024_Slotted_classes_with_slots_in_Python.md) | `class`, `oop`, `special methods`, `dunder methods`, `memory`, `performance`, `dataclasses` |
| [The Art of Scope Management in Modular Python Design](notes/0025_The_Art_of_Scope_Management_in_Modular_Python_Design.md) | `scope`, `sqlalchemy`, `fastapi`, `dependency injection`|
| [Designing Retryable Asynchronous APIs Using functools.partial and Custom Decorators](notes/0026_Designing_Retryable_Asynchronous_APIs_Using_functools_partial_and_Custom_Decorators.md) | `rest api`, `async`, `fastapi`, `functools`, `decorators`, `retry`, `partial` |
| [Dataclasses vs Pydantic vs TypedDict vs NamedTuple in Python](notes/0027_Dataclasses_vs_Pydantic_vs_TypedDict_vs_NamedTuple_in_Python.md) | `class`, `data model`, `dataclass`, `pydantic`, `typeddict`, `namedtuple`, `fastapi`|
| [Why Composition Beats Inheritance in Large-Scale Python Systems](notes/0028_Why_Composition_Beats_Inheritance_in_Large_Scale_Python_Systems.md) | `class`, `inheritance`, `composition`, `fastapi`, `payment`|
| [Encapsulation and Domain-Driven Design in Python Projects](notes/0029_Encapsulation_and_Domain_Driven_Design_in_Python_Projects.md) | `class`, `encapsulation`, `domain driven design (ddd)`, `order`|
| [Instance vs Class vs Static Methods in Python](notes/0030_Instance_vs_Class_vs_Static_Methods_in_Python.md) | `class`, `instance`, `instance method`, `classmethod`, `staticmethod`, `function`|
| [Combining Abstract Classes with Factory and Strategy Patterns in Python](notes/0031_Combining_Abstract_Classes_with_Factory_and_Strategy_Patterns_in_Python.md) | `class`, `abstract class`, `abc`, `abstractmethod`, `payment`, `notification`, `factory class`|
| [Architecting a Multithreaded Log Monitor in Python](notes/0032_Architecting_a_Multithreaded_Log_Monitor_in_Python.md) | `threading`, `gil`, `memory`, `performance`, `queue`, `logging`|
| [Advanced Shared State Management in Python Multiprocessing](notes/0033_Advanced_Shared_State_Management_in_Python_Multiprocessing.md) | `multiprocessing`, `shared states`, `context manager`, `shared memory`|
| [Mastering Task Lifecycle in Python’s asyncio](notes/0034_Mastering_Task_Lifecycle_in_Pythons_asyncio.md) | `asyncio`, `task`, `taskgroup`, `fastapi`|



<br>

## Database & ORM

| Article | Tags | 
|--|--|
| [Designing Robust Transaction Management with Nested Transactions and Savepoints in SQLAlchemy](notes/0035_Designing_Robust_Transaction_Management_with_Nested_Transactions_and_Savepoints_in_SQLAlchemy.md) | `sqlalchemy`, `transaction`, `savepoint`, `nested`, `commit`, `rollback`, `retry`|
| [Optimistic vs. Pessimistic Locking in ORMs](notes/0036_Optimistic_vs_Pessimistic_Locking_in_ORMs.md) | `locking`, `sqlalchemy`, `row-level`, `table-level`, `advisory-level` |
| [Designing Reusable and Scalable ORM Models with Declarative Base and Mixins](notes/0037_Designing_Reusable_and_Scalable_ORM_Models_with_Declarative_Base_and_Mixins.md) | `sqlalchemy`, `declarative base`, `mixin`, `declared_attr`, `registry`, `soft delete`, `uuid`, `timestamp`, `audit`                  |
| [How to Defeat the N+1 Problem with joinedload, selectinload, and subqueryload](notes/0038_How_to_Defeat_the_N1_Problem_with_joinedload_selectinload_and_subqueryload.md) | `sql`, `sqlalchemy`, `relationships`,  `n+1`, `performance`, `query`, `lazy loading`, `joinedload`, `selectinload`, `subqueryload`, `bookstore` |
| [Handling Data in Alembic Migrations When Schema Changes Aren’t Enough](notes/0039_Handling_Data_in_Alembic_Migrations_When_Schema_Changes_Arent_Enough.md)| `sql`, `migration`, `alembic`, `data model`, `schema`, `enum`, `denormalization` |
| [Building Secure and Scalable Multitenant Systems Basics for B2B SaaS](notes/0040_Building_Secure_and_Scalable_Multitenant_Systems_Basics_for_B2B_SaaS.md)| `sql`, `sqlalchemy`, `multitenant`, `saas`, `data schema`, `data model`, `event listener`, `user roles`|
| [Database Schema Design Patterns for Building Scalable E-commerce Applications](notes/0041_Database_Schema_Design_Patterns_for_Building_Scalable_Ecommerce_Applications.md)| `sqlalchemy`, `relationships`, `data schema`, `one-to-many`, `many-to-many`, `normalize`, `denormalize`, `polymorphic`, `e-commerce` |
| [The Art of Not Losing Your Data (or Your Mind) with Isolation Levels](notes/0042_The_Art_of_Not_Losing_Your_Data_or_Your_Mind_with_Isolation_Levels.md) | `sql`, `sqlalchemy`, `isolation`, `acid`, `serializable`, `phantom read` |
| [Explicit vs Implicit Transaction Management in ORMs](notes/0043_Explicit_vs_Implicit_Transaction_Management_in_ORMs.md) | `sqlalchemy`, `transaction`, `session`, `async`, `acid`, `context manager`, `fastapi`, `savepoint`, `nested`, `retry`, `event listener` |
| [The Ultimate Guide to Full Text Search and Filter Implementation with PostgreSQL and SQLAlchemy](notes/0044_The_Ultimate_Guide_to_Full_Text_Search_and_Filter_Implementation_with_PostgreSQL_and_SQLAlchemy.md) | `postgresql`, `sqlalchemy`, `full text search`, `filter`, `sql function`, `TSVector`, `TSQuery`, `__table_args__`, `film`|
| [Connection Pooling Deep Dive with SQLAlchemy](notes/0045_Connection_Pooling_Deep_Dive_with_SQLAlchemy.md) | `sqlalchemy`, `connection pool`, `fastapi`, `QueuePool`, `event listener`, `prometheus_client`, `gunicorn`, `asyncpg`|
| [Managing Bidirectional Relationships in SQLAlchemy with backref and back_populates](notes/0046_Managing_Bidirectional_Relationships_in_SQLAlchemy_with_backref_and_back_populates.md) | `sqlalchemy`, `relationships`, `backref`, `back_populates`, `blog` |

