# The Ultimate Guide to Full Text Search and Filter Implementation with PostgreSQL and SQLAlchemy

Check this post on my blog [here](https://hevalhazalkurt.com/blog/the-ultimate-guide-to-full-text-search-and-filter-implementation-with-postgresql-and-sqlalchemy/).

<br>

Ah, search and filter, the bread and butter of every backend developer's existence. Whether you're building the next IMDb or just trying to find that one obscure 80s sci-fi movie with the talking dolphin, efficient search is crucial.

In the project I was working on, I dealt with product-based data, and we wanted to offer users a comprehensive search and filtering system. Given our complex, highly relational data, this wasn’t exactly a walk in the park. We were handling search and filter queries somehow, but the tangled web of relationships was murdering performance. I’ve lost count of how many queries I’ve inspected with `EXPLAIN ANALYZE` to diagnose and fix performance issues. All I remember is my brain overheating like a cheap laptop and me eventually resigning to my fate as a certified idiot by the end of the day. ****So, I needed to build a more efficient search and filtering system.

I started by creating a dedicated search table stripping away unnecessary data and populating only what was essential for search and filtering. Of course, this was just one piece of the puzzle. Maybe in another post, I’ll break down the entire system, but for now, I’ll use a simplified example to show you how to implement full-text search and filtering.

In this guide, we'll implement a PostgreSQL-backed search system using SQLAlchemy, covering:

- Schema design with optimal indexing (b-tree, GIN, etc.)
- Array sanitization (because nobody likes dirty data)
- Full-text search (TSVector to the rescue)
- Complex filtering (multiple fields, arrays, ranges)
- Performance considerations (indexes, query planning)

Let’s roll!

## Step 1 → Let’s Build Our `FilmSearchIndex` Table

First, we need a table to store our movie data. It’s a movie search index table that optimized for search & filter functionality, not normalized for a classic relational model. It includes titles, genres, names, text blobs, and even a `tsvector` field for full-text search.

```python
from sqlalchemy import Column, Integer, String, Float, Text, ARRAY
from sqlalchemy.dialects.postgresql import TSVECTOR
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class FilmSearchIndex(Base):

    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
    alternate_names = Column(ARRAY(String))  # ["Alternative Title", "International Title"]
    director = Column(String)
    writer = Column(String)
    producer = Column(String)
    release_year = Column(Integer)
    plotline = Column(Text)
    genres = Column(ARRAY(String))  # ["Sci-Fi", "Comedy", "Horror"]
    actors = Column(ARRAY(String))  # ["Bruce Willis", "Gary Oldman"]
    tags = Column(ARRAY(String))  # ["Cult Classic", "Oscar Winner"]
    budget = Column(Float)
    rating = Column(Float)
    search_text = Column(TSVECTOR)  # For full-text search
```

At this stage, properly indexing your data fields is absolutely critical for query performance. Luckily, PostgreSQL offers multiple index types. Here's a battle-tested strategy you can follow:

| **Field** | **Index Type** | **Why?** |
| --- | --- | --- |
| **`id`** | Primary Key (B-tree) | Default, fast lookups |
| **`name`, `director`, `writer`, `producer`** | B-tree | Exact matches, sorting |
| **`release_year`, `budget`, `rating`** | B-tree | Range queries (e.g., **`WHERE year > 1990`**) |
| **`genres`**, **`actors`** | GIN | Optimized for array operations (**`@>`**, **`&&`**) |
| **`search_text`** | GIN | Full-text search acceleration |

```python
from sqlalchemy import Index

class FilmSearchIndex(Base):

    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
    alternate_names = Column(ARRAY(String))  # ["Alternative Title", "International Title"]
    director = Column(String)
    writer = Column(String)
    producer = Column(String)
    release_year = Column(Integer)
    plotline = Column(Text)
    genres = Column(ARRAY(String))  # ["Sci-Fi", "Comedy", "Horror"]
    actors = Column(ARRAY(String))  # ["Bruce Willis", "Gary Oldman"]
    tags = Column(ARRAY(String))  # ["Cult Classic", "Oscar Winner"]
    budget = Column(Float)
    rating = Column(Float)
    search_text = Column(TSVECTOR)  # we'll update for full-text search soon
    
    
    __table_args__ = (
        Index("ix_film_search_index_name", name),
        Index("ix_film_search_index_director", director),
        Index("ix_film_search_index_writer", writer),
        Index("ix_film_search_index_producer", producer),
        Index("ix_film_search_index_release_year", release_year),
        Index("ix_film_search_index_rating", rating),
        Index("ix_film_search_index_budget", budget),
        Index("ix_film_search_index_genres", genres, postgresql_using='gin'),
        Index("ix_film_search_index_actors", actors, postgresql_using='gin'),
        Index("ix_film_search_index_tags", tags, postgresql_using='gin'),
        Index("ix_film_search_index_search_text", "search_text", postgresql_using='gin'),
    ) 
```

## **Step 2 → Create SQL Functions To Sanitize Data**

Before dumping data into **`search_text`**, we need to clean up arrays like cleaning out `NULL`s, removing duplicates, lowercase strings, etc. Because array fields work just fine for filtering queries, we need to handle them carefully when incorporating them into full-text search. There are two key steps to this:

- Sanitizing the data inside arrays
- Converting arrays to strings

```sql
-- sanitize
CREATE OR REPLACE FUNCTION sanitize_array(anyelement) RETURNS anyelement AS $$
  DECLARE
    x text;
    new_arr text[];
  BEGIN
    IF pg_typeof($1) = 'varchar[]'::regtype THEN
      FOREACH x IN ARRAY $1
      LOOP
        new_arr := array_append(new_arr, regexp_replace(x::text, '[[:punct:]]', '', 'g'));
      END LOOP;
    RETURN new_arr;
    ELSEIF pg_typeof($1) = 'varchar'::regtype THEN
      return regexp_replace($1::text, '[[:punct:]]', ' ', 'g');
    END IF;
  return false;
  END;
  $$ LANGUAGE plpgsql IMMUTABLE;
  
  
-- make string
CREATE OR REPLACE FUNCTION immutable_array_to_string(text[], text)
    RETURNS text as $$ SELECT array_to_string($1, $2); 
    $$ LANGUAGE sql IMMUTABLE
```

## **Step 3 → Populating `search_text` with TSVector**

PostgreSQL’s TSVector allows fast full-text search. We’ll concatenate all searchable fields into one `search_text` column. Now let’s update our field.

```python
from sqlalchemy import Index

class FilmSearchIndex(Base):
    ...
    # before
    search_text = Column(TSVECTOR)
    
    # updated
    search_text = Column(
        TSVector(),
        Computed(
            """to_tsvector('english',
                        search_sanitizer(coalesce(name, '')) || ' '
                        || search_sanitizer(coalesce(director, '')) || ' '
                        || search_sanitizer(coalesce(writer, '')) || ' '
                        || search_sanitizer(coalesce(producer, '')) || ' '
                        || immutable_array_to_string(coalesce(search_sanitizer(coalesce(genres, array['{}'])), '{}'), ' ') || ' '
                        || immutable_array_to_string(coalesce(search_sanitizer(coalesce(actors, array['{}'])), '{}'), ' ') || ' '
                        || immutable_array_to_string(coalesce(search_sanitizer(coalesce(tags, array['{}'])), '{}'), ' ') || ' '
                    )""",
            persisted=True,
        ),
        nullable=True,
        index=True,
    )
```

We used **`persisted=True`** here because it stores the computed TSVector on disk instead of recalculating it every query and your database isn’t a treadmill. Trade-off? Slightly slower writes, but lightning-fast searches.

A little warning! If you're using the alembic-pydantic duo with the autogenerate command, don't forget to check your migration file. They're not particularly good at catching changes in computed fields. In such cases, you might need to manually add these changes to your migration script.

## **Step 4 → Inserting Sample Data**

We handled the hard part. Let’s add some classic films to test. 

```python
films = [
    FilmSearchIndex(
        name="The Matrix",
        alternate_names=["Matrix"],
        director="Lana Wachowski",
        genres=["Sci-Fi", "Action"],
        actors=["Keanu Reeves", "Laurence Fishburne"],
        release_year=1999,
        plotline="A hacker discovers a dystopian reality controlled by machines.",
        rating=8.7,
    ),
    FilmSearchIndex(
        name="Inception",
        director="Christopher Nolan",
        genres=["Sci-Fi", "Thriller"],
        actors=["Leonardo DiCaprio", "Tom Hardy"],
        release_year=2010,
        plotline="A thief enters people's dreams to steal secrets.",
        rating=8.8,
    ),
    FilmSearchIndex(
        name="Sharknado",
        director="Anthony C. Ferrante",
        genres=["Disaster", "Comedy", "Horror"],
        actors=["Ian Ziering", "Tara Reid"],
        release_year=2013,
        plotline="Tornadoes fling sharks onto Los Angeles. Chaos ensues.",
        rating=3.3,
    ),
]

session.add_all(films)
session.commit()
```

## **Step 5 → Implementing Search & Filter**

Before querying something let’s first look at PostgreSQL search and filter operators for when you need to find data without losing your sanity. 

### **Full-Text Search (TSVector)**

- **Operator:** **`@@`**
    
    *“The 'Google' of PostgreSQL, if Google hated typos”*
    
    ```sql
    -- SQL
    SELECT * FROM FilmSearchIndex
    WHERE search_text @@ to_tsquery('english', 'dolphin & tornado');
    ```
    
    python
    
    ```python
    # SQLAlchemy
    session.query(FilmSearchIndex).filter(
        FilmSearchIndex.search_text.op("@@")(func.to_tsquery('english', 'dolphin & tornado'))
    )
    ```
    

### **Array Fields**

- **Operator:** **`ANY`**
    
    *“Is Keanu Reeves in this garbage fire of a movie?”*
    
    ```sql
    -- SQL
    SELECT * FROM FilmSearchIndex
    WHERE 'Keanu Reeves' = ANY(actors);
    ```
    
    ```python
    # SQLAlchemy
    session.query(FilmSearchIndex).filter(FilmSearchIndex.actors.any('Keanu Reeves'))
    ```
    
- **Operator:** **`@>`** (Contains)
    
    *“Find films that are both 'Comedy' and ‘Horror’ (aka my life?)”*
    
    ```sql
    -- SQL
    SELECT * FROM FilmSearchIndex
    WHERE genres @> ARRAY['Comedy', 'Horror'];
    ```
    
    ```python
    # SQLAlchemy
    session.query(FilmSearchIndex).filter(FilmSearchIndex.genres.contains(['Comedy', 'Horror']))
    ```
    

### **JSON/JSONB Fields**

- **Operator:** **`>>`**
    
    *“Digging into JSON like it's a Black Friday sale”*
    
    ```sql
    -- SQL
    SELECT * FROM FilmSearchIndex
    WHERE metadata->>'director' = 'Christopher Nolan';
    ```
    
    ```python
    # SQLAlchemy
    session.query(FilmSearchIndex).filter(FilmSearchIndex.metadata['director'].astext == 'Christopher Nolan')
    ```
    

Want to try a little bit more madness?

```python
from sqlalchemy import func, text

# JSON path madness
session.query(FilmSearchIndex).filter(func.jsonb_path_exists(FilmSearchIndex.metadata, '$.awards[*].won'))
```

### **String Fields**

- **Operator:** **`ILIKE`**
    
    *“Case-insensitive search for when you can't spell ‘Benedict Cumberbatch’”*
    
    ```sql
    -- SQL
    SELECT * FROM FilmSearchIndex
    WHERE name ILIKE '%matrix%';
    ```
    
    ```python
    # SQLAlchemy
    session.query(FilmSearchIndex).filter(FilmSearchIndex.name.ilike('%matrix%'))
    ```
    

### **Numeric/Dates**

- **Operator:** **`BETWEEN`**
    
    *“Find films made after the 90s but before our collective hope died”*
    
    ```sql
    -- SQL
    SELECT * FROM FilmSearchIndex
    WHERE release_year BETWEEN 1990 AND 2000;
    ```
    
    ```python
    # SQLAlchemy
    session.query(FilmSearchIndex).filter(FilmSearchIndex.release_year.between(1990, 2000))
    ```
    

## **Step 6 → Testing Results**

Let’s first look at PostgreSQL search and filter operators for when you need to find data without losing your sanity. 

### **1. Full-Text Search (PostgreSQL TSQuery)**

Let’s query to find films matching “hacker dream”:

```python
from sqlalchemy import text

search_query = "hacker dream"
results = session.query(FilmSearchIndex).filter(
    FilmSearchIndex.search_text.op("@@")(func.to_tsquery('english', search_query))
).all()

# Returns: The Matrix (matches "hacker") + Inception (matches "dream")
```

### **2. Filtering by Genre (Array Operations)**

Now let’s find all Sci-Fi films:

```python
results = session.query(FilmSearchIndex).filter(
    FilmSearchIndex.genres.contains(["Sci-Fi"])
).all()

# Returns: The Matrix, Inception
```

### **3. Combined Search + Filter**

Let’s try combine multiple conditions and find Sci-Fi films with ratings > 8.5:

```python
results = session.query(FilmSearchIndex).filter(
    FilmSearchIndex.genres.contains(["Sci-Fi"]),
    FilmSearchIndex.rating > 8.5
).all()

# Returns: Inception (8.8)
```

### **4. Partial Name Search (ILIKE + Index)**

What about partial searches?

```python
results = session.query(FilmSearchIndex).filter(
    FilmSearchIndex.name.ilike("%matrix%")
).all()

# Returns: The Matrix
```

## **Closing Credits**

And there you have it, your PostgreSQL + SQLAlchemy search/filter toolkit is now fully loaded. Whether you’re hunting for cult classics or debugging performance nightmares, these operators will save you from writing SQL that looks like a cry for help.

Of course in this article, we've implemented a simple search and filtering system for clarity. In real-world applications, you'll be dealing with much more complex structures and running sophisticated queries, but once you properly understand the fundamentals, everything else becomes easier to handle.

Remember:

- GIN indexes are the unsung heroes of array/json/search chaos.
- `EXPLAIN ANALYZE` is your therapist when queries go rogue.
- Sharknado is technically a documentary about database migrations gone wrong.

Now go forth and query fearlessly but maybe keep `pg_dump` on speed dial.
