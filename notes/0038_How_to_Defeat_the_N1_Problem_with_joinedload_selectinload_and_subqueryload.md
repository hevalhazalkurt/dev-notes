# How to Defeat the N+1 Problem with `joinedload`, `selectinload`, and `subqueryload`

Check this post on my blog [here](https://hevalhazalkurt.com/blog/how-to-defeat-the-n1-problem-with-joinedload-selectinload-and-subqueryload/).

<br>

If you're working on backend systems with SQLAlchemy, one of the best ways to make your app fast and efficient is by understanding how data is loaded from the database. The difference between a snappy API and a sluggish one often boils down to a few lines of code where SQLAlchemy is silently loading your data. That's where eager loading comes in.

In this deep dive, we'll explore how SQLAlchemy loads related objects using:

- `lazy` (the default)
- `joinedload`
- `selectinload`
- `subqueryload`

We'll start from the very basics, set up a clear data model, and then move into advanced patterns, real-life examples, and how these loading strategies affect performance and architecture.

## Understanding the Basics

Before diving into eager loading, let’s get grounded on the core concepts that influence how SQLAlchemy handles data relationships:

### What is a Relationship?

In SQLAlchemy, a relationship connects rows from one table to rows in another. For example, a book is written by an author, and that relationship allows you to access `book.author` in Python code. SQLAlchemy manages these links so that you can navigate between related records like objects in Python.

### Lazy Loading (Default Behavior)

SQLAlchemy doesn’t fetch related data until you try to access it (if you don’t define the relationship loading on your model class). This is lazy loading, and it's the default. It keeps queries simple but can result in a large number of queries when accessing related objects in loops or chains.

```python
book = session.query(Book).first()
print(book.author.name)  # Triggers a second query
```

### N+1 Problem

The N+1 problem happens when your code triggers one query to fetch N rows, and then N more queries to fetch related data for each of those rows.

```python
books = session.query(Book).all()
for book in books:
    print(book.author.name)  # 1 + N queries!
```

This can crush performance and database throughput. Eager loading helps prevent this.

### Eager Loading

Eager loading tells SQLAlchemy: “Also load this related table right now, because I know I’ll need it.” It's a proactive way to get related data in fewer queries.

## Base Data Model → Bookstore Example

We'll use a simplified bookstore schema to illustrate loading strategies throughout the blog. This model has nested relationships that are common in real backend systems.

```python
class Publisher(Base):
    __tablename__ = 'publishers'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    books = relationship("Book", back_populates="publisher")

class Author(Base):
    __tablename__ = 'authors'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    books = relationship("Book", back_populates="author")

class Book(Base):
    __tablename__ = 'books'
    id = Column(Integer, primary_key=True)
    title = Column(String)
    publisher_id = Column(Integer, ForeignKey('publishers.id'))
    author_id = Column(Integer, ForeignKey('authors.id'))

    publisher = relationship("Publisher", back_populates="books")
    author = relationship("Author", back_populates="books")
    reviews = relationship("Review", back_populates="book")

class Review(Base):
    __tablename__ = 'reviews'
    id = Column(Integer, primary_key=True)
    content = Column(String)
    book_id = Column(Integer, ForeignKey('books.id'))
    book = relationship("Book", back_populates="reviews")
```

## Lazy Loading (The Default)

Lazy loading loads related data on access. It's simple to use but dangerous in loops.

```python
books = session.query(Book).all()

for book in books:
    print(book.author.name)  # Triggers a separate query per author
```

If you have 100 books, this results in 101 queries: 1 for books, and 100 for authors. This is the classic N+1 problem. Lazy loading is okay when you're only fetching one or two objects, but in lists or batch operations, it's inefficient.

## `joinedload` → The Inline JOIN Option

`joinedload` uses a SQL `JOIN` to fetch related objects in one query.

```python
from sqlalchemy.orm import joinedload

books = (
    session.query(Book)
    .options(joinedload(Book.author))
    .all()
)
```

SQL:

```sql
SELECT books.*, authors.*
FROM books
LEFT OUTER JOIN authors ON authors.id = books.author_id
```

This is great for fetching related objects without extra queries. But it comes with a risk: if you also join more relationships like publisher and reviews, the result set can get huge due to row multiplication.

```python
books = session.query(Book).options(
    joinedload(Book.author),
    joinedload(Book.publisher),
    joinedload(Book.reviews)
).all()
```

If a book has 5 reviews, this query will repeat that book row 5 times. It’s efficient for small or flat relationships, but not for large collections.

## `selectinload` → Separate `WHERE IN` Queries

`selectinload` issues a second query to get related records, using a `WHERE IN` clause.

```python
from sqlalchemy.orm import selectinload

books = (
    session.query(Book)
    .options(selectinload(Book.reviews))
    .all()
)
```

simplified SQL:

```sql
SELECT * FROM books;
SELECT * FROM reviews WHERE book_id IN (1, 2, 3...);
```

This avoids row explosion by not using a `JOIN`. It's ideal for one-to-many relationships like `Book.reviews` where you don’t want to repeat book rows for each review. Imagine a dashboard listing books with a count of reviews. Using `joinedload` would inflate rows. But `selectinload` keeps books cleanly separated and pulls reviews efficiently.

## `subqueryload` → Use Subqueries

`subqueryload` behaves like `selectinload` but uses a subquery instead of a `WHERE IN`. This is better when dealing with filtered or paginated data.

```python
from sqlalchemy.orm import subqueryload

books = (
    session.query(Book)
    .options(subqueryload(Book.reviews))
    .all()
)
```

Simplified SQL:

```sql
SELECT * FROM books;
SELECT * FROM reviews WHERE book_id IN (SELECT id FROM books);
```

While the end result is similar to `selectinload`, `subqueryload` performs better when you're dealing with large datasets, complicated joins, or when the database planner can optimize subqueries better than long `IN` lists.

## Combining Strategies

In real-world apps, you often need to load several relationships at once. Using a mix of strategies lets you balance performance and readability.

```python
books = session.query(Book).options(
    joinedload(Book.author),         # Many-to-one
    selectinload(Book.reviews),      # One-to-many
    joinedload(Book.publisher)       # Many-to-one
).all()
```

This avoids the N+1 problem, prevents cartesian explosions, and keeps queries manageable. Always choose the strategy based on relationship type and data size.

## Summary Table

| **Relationship Type** | **Recommended Strategy** |
| --- | --- |
| Many-to-one (Book → Author) | `joinedload` |
| One-to-many (Book → Reviews) | `selectinload` |
| Deep nesting or filters | `subqueryload` |
| Rare access | `lazy` |

## Common Pitfalls

- **Cartesian explosion →** Using `joinedload` on deep or large collections like both `author` and `reviews`.
- **N+1 trap →** Forgetting to use eager loading for nested relationships.
- **Wrong combinations →** Using `joinedload` for large lists that should be separate.

## Final Thoughts

Eager loading is a key part of writing scalable backend systems. Understanding when to use `joinedload`, `selectinload`, or `subqueryload` can drastically improve performance. In real-world APIs, especially dashboards and detail views, it's common to fetch deeply nested data  and if you're not careful, your server could execute hundreds of queries unnecessarily.