# Django ORM Phase 1 Notes

## 1. What is an ORM?

ORM stands for **Object Relational Mapper**.

It is a layer between your Python code and a relational database
(PostgreSQL, MySQL, SQLite).

Its job is to translate Python operations into SQL and convert database
rows back into Python objects.

Flow:

``` text
Python Code
    │
    ▼
Django ORM
    │
    ▼
SQL
    │
    ▼
Database
    │
    ▼
Rows
    │
    ▼
Python Objects
```

Example:

``` python
Book.objects.filter(price__gt=20)
```

Generates SQL similar to:

``` sql
SELECT * FROM book WHERE price > 20;
```

### Why use an ORM?

-   Write Python instead of raw SQL.
-   Safer and more readable.
-   Database-independent.
-   Maps database rows to Python objects.

> The ORM does **not** store data. The database stores data. The ORM
> only translates.

------------------------------------------------------------------------

## 2. Models

A model is a Python class that:

1.  Defines the database table structure.
2.  Represents rows from that table as Python objects.

Example:

``` python
class Book(models.Model):
    title = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=6, decimal_places=2)
```

Migration creates a table similar to:

  id   title      price
  ---- -------- -------
  1    Django     25.00

### Important

`Book` is **not** the table.

`Book` is a Python class.

An instance:

``` python
book = Book.objects.get(id=1)
```

represents one row.

------------------------------------------------------------------------

## 3. Manager (`objects`)

Every model gets a default manager called `objects`.

It is the entry point for database operations.

Examples:

``` python
Book.objects.all()
Book.objects.filter(...)
Book.objects.get(...)
Book.objects.create(...)
```

Think of the manager as:

> "The object responsible for building QuerySets and communicating with
> the database."

------------------------------------------------------------------------

## 4. QuerySet

A QuerySet is **not** the data.

It is a blueprint describing the SQL Django should execute later.

Example:

``` python
books = Book.objects.filter(price__gt=20)
```

No SQL runs yet.

`books` only stores the query definition.

------------------------------------------------------------------------

## 5. Lazy Evaluation

Django delays executing SQL until the data is actually needed.

``` python
books = Book.objects.filter(price__gt=20)
```

No SQL.

SQL runs when evaluated:

``` python
for book in books:
    print(book.title)

books.first()
books.exists()
books.count()
len(books)
print(books)
```

Reason:

-   Avoid unnecessary queries.
-   Combine multiple filters into one SQL statement.
-   Improve performance.

------------------------------------------------------------------------

## 6. QuerySet Cache

After evaluation, Django stores returned model instances in the QuerySet
cache.

Example:

``` python
books = Book.objects.filter(price__gt=20)

for b in books:
    print(b.title)

for b in books:
    print(b.price)
```

Only one SELECT query executes.

The second loop uses cached objects.

### count() vs cache

``` python
books.count()
```

before evaluation performs:

``` sql
SELECT COUNT(*) ...
```

and does **not** populate the object cache.

If the QuerySet has already been fully evaluated, `count()` can use the
cached objects instead of another COUNT query.

------------------------------------------------------------------------

## 7. Creating Model Objects

### Method 1

``` python
book = Book(title="Python")
```

Creates only a Python object.

SQL executed?

**No**

------------------------------------------------------------------------

### Method 2

``` python
book.save()
```

Synchronizes the object with the database.

If `id is None`:

``` sql
INSERT
```

------------------------------------------------------------------------

### Method 3

``` python
Book.objects.create(title="Python")
```

Equivalent to:

``` python
book = Book(title="Python")
book.save()
```

------------------------------------------------------------------------

## 8. INSERT vs UPDATE

### New object

``` python
book = Book(title="Python")
```

State:

    id = None

Calling:

``` python
book.save()
```

executes:

``` sql
INSERT
```

Database assigns a primary key.

------------------------------------------------------------------------

### Existing object

``` python
book = Book.objects.get(id=1)
book.title = "Advanced Python"
book.save()
```

Because the object already has a primary key, Django executes:

``` sql
UPDATE book
SET title='Advanced Python'
WHERE id=1;
```

Rule:

-   id is None → INSERT
-   id exists → UPDATE

------------------------------------------------------------------------

## 9. Python Object vs Database Row

Changing attributes only changes the Python object.

``` python
book.title = "Advanced Python"
```

Database is unchanged.

Only after:

``` python
book.save()
```

does the database update.

Mental model:

    Python Object
          │
    Change attributes
          │
          ▼
    Only memory changes
          │
    save()
          │
          ▼
    Database changes

------------------------------------------------------------------------

## Interview Questions

1.  What is an ORM?
2.  Why can't PostgreSQL execute `Book.objects.all()`?
3.  What is the difference between a model and a database table?
4.  What is a QuerySet?
5.  Explain lazy evaluation.
6.  When does SQL execute?
7.  What is QuerySet caching?
8.  Difference between `Book()` and `Book.objects.create()`.
9.  Difference between `save()` on a new and existing object.
10. Difference between `len(queryset)` and `queryset.count()`.

## Key Takeaways

-   ORM translates Python ↔ SQL.
-   Models define tables and represent rows.
-   `objects` is the model manager.
-   QuerySets are lazy.
-   QuerySets cache evaluated objects.
-   `Book()` does not touch the database.
-   `save()` synchronizes with the database.
-   `id is None` → INSERT.
-   `id exists` → UPDATE.
-   Changing attributes does not change the database until `save()` is
    called.
