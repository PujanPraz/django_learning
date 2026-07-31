# Django ORM — Complete Detailed Notes

## 1. What is an ORM?

**ORM = Object Relational Mapper.**

Databases like PostgreSQL only understand SQL. They have no idea what a Python class or object is. But writing raw SQL for every operation in a large application is tedious, repetitive, and error-prone.

The ORM sits *between* your Python code and the database, and acts as a translator in both directions:

```
Python ORM code  --->  SQL  --->  PostgreSQL
Python Model Objects <--- Rows <--- PostgreSQL
```

So when you write:

```python
Book.objects.all()
```

Django internally converts this into something like:

```sql
SELECT * FROM book;
```

sends it to PostgreSQL, gets back raw rows, and converts each row back into a `Book` Python object you can work with normally (`book.title`, `book.price`, etc.).

**Why this matters:** you get to think in terms of Python objects and relationships instead of SQL syntax, joins, and string-building — while Django handles the translation, escaping, and optimization underneath.

---

## 2. Models — Python Representation of Tables

A **model** is a Python class that represents a single database table. Each **field** on the model represents a **column**.

```python
class Book(models.Model):
    title = models.CharField(max_length=100)
```

This tells Django: "create a table called `book` with a `title` column that holds text up to 100 characters." Django also auto-adds an `id` primary key column unless you specify otherwise.

**Key idea:** the model is the *schema*, not the data. It defines what a row *can* contain, not any particular row itself.

---

## 3. Model Instances — Individual Rows

An **instance** of a model is one Python object representing one row.

```python
book = Book(title="Django")
```

At this point:
- A Python object exists in memory.
- **No SQL has run.**
- **Nothing is saved** to the database yet.

Think of this like filling out a paper form but not yet submitting it — the data exists on your desk, not in the filing cabinet.

### Saving

```python
book.save()
```

This is the moment SQL actually executes:
- If the object has **no primary key yet**, Django knows this is a new row and issues an `INSERT`.
- Once the row is inserted, PostgreSQL generates a primary key (e.g., `id=1`), and Django **assigns that key back onto the Python object** (`book.id` now equals `1`).
- If you call `.save()` **again** on the same object, Django sees it already has a primary key, so it issues an `UPDATE` instead of a new `INSERT` — meaning it modifies the existing row rather than creating a duplicate.

**Golden rule:** presence of a primary key on the instance is what decides INSERT vs UPDATE.

---

## 4. Managers — Entry Point to the Table

```python
Book.objects
```

`objects` is called a **Manager**. It's not tied to any one row — it's tied to the *whole table*. It's the starting point for any table-wide operation:

| Method | Purpose |
|---|---|
| `all()` | get every row |
| `filter()` | get rows matching conditions |
| `create()` | build + save a row in one step |
| `get()` | fetch exactly one row (errors if 0 or more than 1 found) |

**Mental model:** Model = table blueprint. Manager (`objects`) = the "front desk" you go to when you want to query that table.

---

## 5. QuerySets — Queries, Not Data

This is one of the most important — and most misunderstood — concepts in Django.

```python
books = Book.objects.filter(price__gt=100)
```

`books` here is **not a list of Book objects**. It is a **QuerySet** — a Python object that represents a *description* of a database query that hasn't run yet.

Internally, you can imagine it storing something like:

```
Model  = Book
Filter = price > 100
```

No SQL has executed at this line. It's essentially a "recipe" for a query, not the query's results.

### Chaining

Because a QuerySet is just a description, you can keep modifying it:

```python
books = Book.objects.filter(price__gt=100)
books = books.filter(stock__gt=0)
books = books.order_by("title")
```

Each step adds to the recipe — it does **not** touch the database, and does **not** create new data each time.

### When does SQL actually run?

SQL executes only when Django needs actual data — this is called **evaluation**. Common triggers:

- `first()`
- `get()`
- `count()`
- `exists()`
- iterating over it (`for b in books:`)
- `list(books)`
- `len(books)`

---

## 6. Lazy Evaluation

Django deliberately **delays** running SQL for as long as possible. This is called **lazy evaluation**.

```python
Book.objects.filter(price__gt=100)          # No SQL yet
Book.objects.filter(price__gt=100).first()  # SQL executes now
```

### Why is this useful?

1. **Query optimization** — Django can combine multiple `.filter()` calls into a single efficient SQL statement instead of running each one separately.
2. **Query chaining** — you can build up complex queries step by step (e.g., across different functions or conditional logic) without wasting database calls along the way.
3. **Fewer database hits** — the database is only contacted once you actually need the result, not every time you add a condition.

**Analogy:** it's like building a shopping list — you keep adding/removing items, but you only make the actual trip to the store once, when you're ready to shop.

---

## 7. QuerySet Caching

Once a QuerySet is evaluated (SQL has run and results are fetched), Django **caches** those results inside the QuerySet object.

```python
books = Book.objects.filter(price__gt=100)
list(books)   # SQL runs, results cached
list(books)   # Uses cache — no new SQL
```

**Important nuance:** this cache is tied to that *specific* QuerySet object. If you write the same query again as a fresh expression, it will hit the database again:

```python
Book.objects.filter(price__gt=100)  # fresh QuerySet -> new SQL if evaluated
```

---

## 8. ForeignKey — One-to-Many Relationships

```python
class Expense(models.Model):
    group = models.ForeignKey(Group, related_name="expenses", on_delete=models.CASCADE)
```

A `ForeignKey` column stores the **primary key of a related row** in another table. Here, each `Expense` row stores the `id` of the `Group` it belongs to.

### Why use it?
- **Removes duplicated data** — instead of copying group details into every expense row, you just reference the group's ID.
- **Maintains consistency** — if group info changes, you don't have to update it in a hundred places.
- **Creates relationships** — Django can now traverse from expenses to groups and back.

### Forward access (many-to-one direction)

```python
expense.group
```

Returns a **single** `Group` object — the one this expense belongs to.

### Reverse access (one-to-many direction)

```python
group.expenses.all()
```

Returns **all** `Expense` objects linked to that group. This works because of `related_name="expenses"` defined in the ForeignKey — it creates this reverse accessor.

---

## 9. Related Managers

```python
group.expenses
```

This is called a **Related Manager**. It behaves almost identically to `Expense.objects`, with one key difference: it's **automatically filtered** to only that group's expenses. So:

```python
group.expenses.all()      # only this group's expenses
group.expenses.filter(...) # filtering within this group's expenses
```

is conceptually the same toolkit as `Expense.objects`, just pre-scoped.

---

## 10. The N+1 Query Problem

This is a classic performance trap.

```python
expenses = Expense.objects.all()

for e in expenses:
    print(e.group.name)
```

What happens under the hood:

- **1 query** to fetch all expenses.
- Then, for **each** expense in the loop, accessing `e.group.name` triggers **another separate query** to fetch that expense's group (because `group` wasn't pre-loaded).

If there are `N` expenses, that's:

```
1 (expenses) + N (one per group lookup) = 1 + N queries
```

**Why it's a problem:** if you have 1,000 expenses, this fires up to 1,001 separate database queries instead of a small, efficient number — massively slowing down the application.

---

## 11. select_related() — Fixing N+1 for Forward/One-to-One Relations

```python
Expense.objects.select_related("group")
```

- Used for **ForeignKey** and **OneToOne** relationships.
- Internally uses an SQL **JOIN** to fetch the expense *and* its related group **in a single query**.
- Best suited when each object relates to **exactly one** other object (many-to-one or one-to-one), because a JOIN naturally returns one combined row per match.

This eliminates the N+1 problem for forward relationships in one shot.

---

## 12. Many-to-Many Relationships

Example: many students can take many courses, and many courses can have many students.

This can't be represented with a simple foreign key column, because neither side has just "one" of the other. Instead, relational databases use a **junction table** (also called an association table):

```
Student        Course        Student_Course
---------      --------      ---------------
id             id            student_id
name           title         course_id
```

The junction table just stores pairs of IDs, linking specific students to specific courses.

In Django, you don't need to build this table by hand — declaring:

```python
courses = models.ManyToManyField(Course)
```

makes Django automatically create and manage that junction table behind the scenes.

---

## 13. prefetch_related() — Fixing N+1 for Many-to-Many / Reverse FK

```python
Group.objects.prefetch_related("expenses")
```

- Used for **ManyToMany** fields and **reverse ForeignKey** relationships (like `group.expenses`).
- Unlike `select_related()`, this does **not** use a single JOIN. A JOIN here would cause **duplicated parent rows** (e.g., a Group with 5 expenses would appear 5 times if joined directly).
- Instead, Django runs it as **two separate, efficient queries**:
  1. Query 1: fetch all the `Group` rows.
  2. Query 2: fetch all the related `Expense` rows for those groups.
- Then Django stitches these together **in Python** (not in SQL), attaching the right expenses to the right groups.

### Rule of thumb

| Relationship type | Method to use |
|---|---|
| ForeignKey / OneToOne (single related object) | `select_related()` |
| Reverse ForeignKey / ManyToMany (multiple related objects) | `prefetch_related()` |

---

## 14. Q Objects — Complex Conditions (OR, NOT, Grouping)

```python
from django.db.models import Q
```

By default, chaining multiple conditions inside a single `.filter()` call means **AND**:

```python
Book.objects.filter(price__gt=500, stock__gt=0)
```
→ "price > 500 **AND** stock > 0"

But what if you need **OR**, **NOT**, or nested/grouped logic? That's what `Q` objects are for.

```python
Q(price__gt=500) | Q(stock__gt=0)   # OR
Q(price__gt=500) & Q(stock__gt=0)   # AND (explicit)
~Q(stock=0)                          # NOT
```

**Mental model:** each `Q(...)` represents one standalone condition that you can combine using logical operators (`|`, `&`, `~`), similar to combining boolean expressions in plain Python — except these get translated into SQL `WHERE` clause logic.

---

## 15. F Expressions — Database-Side Field Calculations

```python
from django.db.models import F
```

### The problem without F

A naive stock decrement might look like:

```python
product = Product.objects.get(id=1)
product.stock = product.stock - 1
product.save()
```

This requires:
1. A `SELECT` to read the current value into Python.
2. Python does the subtraction.
3. An `UPDATE` to write it back.

That's **two queries**, and worse — if two requests run this at the same time, both might read the same starting value before either writes back, causing a **race condition** where one update effectively gets lost (e.g., stock should have decreased by 2, but only decreases by 1).

### With F

```python
Product.objects.filter(id=1).update(
    stock=F("stock") - 1
)
```

This tells the **database itself** to compute:

```sql
UPDATE product SET stock = stock - 1 WHERE id = 1;
```

### Advantages
- **One SQL query** instead of two.
- **Faster**, since there's no round trip to fetch the value into Python first.
- **Prevents lost updates / race conditions**, because the subtraction happens atomically inside the database itself, not based on a possibly-stale value read earlier in Python.

**Rule of thumb:** whenever a calculation can be done by the database directly (increment, decrement, comparing two fields, etc.), use `F()` instead of pulling data into Python first.

---

## 16. aggregate() — Many Rows → One Result

Used when you want to **collapse an entire QuerySet into a single summary value**.

```python
Expense.objects.aggregate(total=Sum("amount"))
Expense.objects.aggregate(avg=Avg("amount"))
```

- Returns a **dictionary**, e.g. `{"total": 4500}`.
- The database performs the computation directly using functions like `SUM`, `COUNT`, `AVG`, `MIN`, `MAX` — far more efficient than pulling every row into Python and summing manually.

**Use case:** "What's the total of all expenses?" — one number, not one number per row.

---

## 17. annotate() — Many Rows → Many Rows, Each With Extra Computed Data

Used when you want to **keep every row**, but attach an extra calculated field to each one.

```python
Group.objects.annotate(
    expense_count=Count("expenses")
)
```

Now, **each** `Group` object in the result has a new attribute:

```python
group.expense_count
```

showing how many expenses belong to that specific group.

Another example — sum per group instead of overall:

```python
Group.objects.annotate(
    total_amount=Sum("expenses__amount")
)
```

This gives each group its own `total_amount`, computed just for that group's expenses (not the whole table).

### aggregate() vs annotate() — the key difference

| Function | Behavior |
|---|---|
| `aggregate()` | Many rows → **one** result (a single dictionary/summary) |
| `annotate()` | Many rows → **many rows**, each with an extra calculated field attached |

**Analogy:** `aggregate()` is like asking "what's the class average?" (one number). `annotate()` is like asking "what's each student's average across their own tests?" (one number *per student*, but still a list of students).

---

## Golden Rules — Quick Reference

| Concept | Represents |
|---|---|
| Model | Database table (schema) |
| Instance | One row |
| Manager (`objects`) | Entry point to query a table |
| QuerySet | A database query description — not data itself |
| Q object | A single query condition (combinable via `\|`, `&`, `~`) |
| F object | A database-side field calculation |

### Am I building a query, or asking for data?

**Building** (no SQL runs yet — just shaping the query):
- `filter()`
- `exclude()`
- `all()`
- `order_by()`

**Asking** (SQL executes now):
- `get()`
- `first()`
- `count()`
- `exists()`
- iteration (`for x in queryset`)
- `list(queryset)`

This "build vs ask" distinction is the core mental model for understanding *when* Django actually talks to the database — and it's the key to writing efficient, predictable ORM code.
