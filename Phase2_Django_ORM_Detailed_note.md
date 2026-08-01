# Django ORM Notes — Day 2 (Detailed)

This continues from the Day 1 notes, focusing on **efficient bulk operations**, **existence checks**, and **concurrency-safe updates**.

---

## 1. `exists()` — Fast Existence Checks

```python
Book.objects.filter(price__gt=100).exists()
```

### What it does
Returns a plain `True` or `False` — nothing else.

### Why it's faster than `first()`

If you only need to know *whether* matching rows exist (not the actual data), `exists()` is the right tool, because of how it's implemented under the hood:

- `exists()` generates a very lightweight SQL query — essentially "does at least one row matching this condition exist?" (implemented efficiently by the database, often stopping at the first match found, without pulling any column data).
- `first()`, on the other hand, actually **fetches a full row** (all columns) from the database, then builds a full Python model object out of it — even though you might only want the True/False answer.

**Rule of thumb:**
- Need to know *if* something exists → `exists()`.
- Need the actual object/data → `first()` or `get()`.

```python
# Bad: fetches a whole object just to check existence
if Book.objects.filter(price__gt=100).first():
    ...

# Good: only asks the database for a yes/no answer
if Book.objects.filter(price__gt=100).exists():
    ...
```

---

## 2. `update()` — Direct Bulk SQL UPDATE

```python
Book.objects.filter(stock=0).update(is_available=False)
```

### What it does
Runs a **single SQL `UPDATE` statement** directly against the database for every row matching the filter.

### Key characteristics

- **No Python model instances are created.** Django never loads the matching rows into memory as `Book` objects — it just tells the database "update all rows matching this filter" directly.
- **`save()` is never called.** Because there are no model instances involved, none of the usual instance-level behavior tied to `.save()` — like custom `save()` overrides, `pre_save`/`post_save` signals, or `auto_now` field updates — gets triggered.
- **Best used when every matching row should get the exact same new value or the same formula-based change.**

```python
# Same value for every matching row
Book.objects.filter(stock=0).update(is_available=False)

# Same formula applied to every matching row (combine with F())
Book.objects.filter(category="clearance").update(price=F("price") * 0.8)
```

### When NOT to use it

If different rows need **different** final values based on individual logic, `update()` can't handle that in one call — you'd need `bulk_update()` instead (see below), or a loop with individual `.save()` calls if custom save logic must run per object.

---

## 3. `bulk_create()` — Efficient Bulk Inserts

### The problem it solves

Normally, creating many objects one at a time looks like this:

```python
for data in data_list:
    Book.objects.create(**data)   # one INSERT per row — N separate queries
```

This means **N separate database round trips** for N objects — slow at scale.

### The efficient way

```python
books = [Book(title=d["title"], price=d["price"]) for d in data_list]
Book.objects.bulk_create(books)
```

### What happens here

1. First, you build the `Book(...)` objects **in plain Python** — this step is identical to before, and still does **not** hit the database (just like any unsaved instance).
2. Then, `bulk_create()` sends them to the database in a much more efficient way — typically as a **single bulk `INSERT`** statement (or a small number of *batched* INSERTs if the list is very large), instead of one INSERT per object.

### Key characteristics

- **`save()` is never called** on any of the individual objects — this is a fundamentally different, more efficient database code path.
- On databases like **PostgreSQL**, after the bulk insert, Django *can* populate the auto-generated primary keys (`id`) back onto each of the returned Python objects — so after calling `bulk_create()`, your objects can already have their real IDs, letting you continue using them without a separate re-fetch. (This capability depends on the database backend — PostgreSQL supports it well.)

### When to use it
Any time you're inserting **many** rows at once and don't need any custom per-object `save()` logic (like signals or overridden save behavior) to run.

---

## 4. `bulk_update()` — Efficient Bulk Updates (Different Values Per Object)

```python
books = Book.objects.filter(category="fiction")

for book in books:
    book.price = book.price * 1.1   # different new value per object

Book.objects.bulk_update(books, ['price'])
```

### What it does
Updates **many existing model instances** efficiently, where **each object may have a completely different new value**.

### Key characteristics

- You **must specify which fields to update**, as a list — e.g. `['price', 'stock']`. This tells Django exactly which columns to include in the generated SQL, rather than updating every field on every object.
- Like `bulk_create()`, this **does not call `save()`** on each object — it's a separate, more efficient bulk code path that skips per-instance save machinery.
- **Best suited when every object's new value is different** — computed individually in Python beforehand — as opposed to one shared value/formula for all rows.

```python
Book.objects.bulk_update(books, ['price', 'stock'])
```

---

## 5. `update()` vs `bulk_update()` — The Core Distinction

| | `update()` | `bulk_update()` |
|---|---|---|
| Value applied | **Same** value or formula for **all** matching rows | **Different** value **per object**, computed individually |
| SQL generated | One simple `UPDATE ... WHERE ...` | Efficient bulk update (still avoids per-row individual queries) |
| Needs existing Python objects first? | No — works purely on a QuerySet filter | Yes — you typically load objects, modify them in Python, then bulk_update |
| Typical use | "Set all clearance items to `is_available=False`" | "Recalculate a personalized discount for every book individually" |

**Simple test to decide which to use:** *Would this be one single SQL statement that applies the same rule to every row? → `update()`. Does each row need its own distinct computed value based on individual logic? → `bulk_update()`.*

---

## 6. `F()` vs `bulk_update()` — Don't Confuse Them

Both can be used to "update many rows," but they solve **different shaped problems**:

- **`F()`** is for when the **database itself** can compute the new value using a formula that's the *same shape* for every row — e.g., "add 100 to whatever the current price is," applied via `update()`:

  ```python
  Book.objects.filter(category="new").update(price=F("price") + 100)
  ```
  This runs entirely inside the database, in one query, and doesn't need Python to know each row's current value at all.

- **`bulk_update()`** is for when each object's **final value is different and was already computed in Python** — not a formula the database can apply uniformly:

  ```python
  for book in books:
      book.price = calculate_custom_price(book)  # unique logic per book

  Book.objects.bulk_update(books, ['price'])
  ```

**Rule of thumb:**
- If the new value can be expressed as *one formula applied identically to every row* → use `F()` (usually paired with `.update()`).
- If every object needs its **own distinct value**, computed individually → use `bulk_update()`.

---

## 7. `select_for_update()` — Row Locking to Prevent Race Conditions

### The problem

Sometimes you need to **read a value, make a decision based on it, and then update it** — and that "read → decide → update" sequence isn't safe if two requests can run it at the same time.

Example: checking if there's enough stock before decreasing it.

```python
product = Product.objects.get(id=1)
if product.stock > 0:
    product.stock -= 1
    product.save()
```

If two requests run this **simultaneously**, both might read `stock = 1` at the same moment (before either has saved), both decide "yes, stock is available," and both proceed to decrement — resulting in stock going negative, which shouldn't be possible. This is a **race condition** rooted in decision-making logic, not just a plain arithmetic issue (which is why plain `F()` alone can't fully solve this particular case — the *decision* `if product.stock > 0` also needs to be based on trustworthy, locked data).

### The fix

```python
from django.db import transaction

with transaction.atomic():
    product = Product.objects.select_for_update().get(id=1)
    if product.stock > 0:
        product.stock -= 1
        product.save()
```

### What `select_for_update()` does

- It **must be used inside `transaction.atomic()`** — row locking only makes sense within the boundaries of a transaction.
- It tells the database to **lock the selected row(s)** as soon as they're fetched, and hold that lock until the transaction finishes (COMMIT or ROLLBACK).
- While locked, if **another transaction** tries to fetch the *same* row with `select_for_update()`, it will be forced to **wait** until the first transaction finishes — instead of both transactions reading the same stale value and both proceeding incorrectly.

This effectively serializes the "read → decide → update" sequence for that specific row, so two concurrent requests can no longer both see `stock=1` and both decide to proceed — the second one will wait until the first has finished (and committed its updated stock value) before it even gets to read the row.

---

## 8. `F()` vs `select_for_update()` — Different Problems, Both About Concurrency

| | `F()` | `select_for_update()` |
|---|---|---|
| Solves | Safe **database-side arithmetic** on a value (no Python round trip needed) | Safe **decision-making** that depends on reading the current value *before* deciding what to do |
| Needs a transaction? | Not strictly required (though often combined with one) | **Yes**, must be inside `transaction.atomic()` |
| Example use case | "Decrease stock by 1, whatever it currently is" (no conditional check needed) | "Only decrease stock by 1 **if** stock is currently greater than 0" (requires reading & branching on the value first) |

**Rule of thumb:**
- If you're just applying a calculation and don't need to *branch* on the current value first → `F()` is simpler and sufficient.
- If your logic needs to **check** the current value and make a decision **before** updating (like validating there's enough stock, enough balance, etc.) → you need `select_for_update()` to make that read-then-decide sequence safe from concurrent interference.

---

## Quick Summary Table

| Method | Purpose |
|---|---|
| `exists()` | Fast True/False existence check without loading full objects |
| `update()` | Direct bulk SQL update — same value/formula applied to all matching rows |
| `bulk_create()` | Efficiently insert many new objects in one (or few) queries |
| `bulk_update()` | Efficiently update many existing objects that each have different values |
| `F()` | Let the database perform a calculation directly, avoiding read-then-write round trips |
| `select_for_update()` | Lock rows during a transaction to prevent race conditions in read → decide → update logic |
| `transaction.atomic()` | Group multiple operations into one all-or-nothing unit |

### The big-picture mental model

- **Reading efficiently:** `exists()` when you just need yes/no.
- **Writing efficiently, same value everywhere:** `update()` (optionally with `F()` for database-side math).
- **Writing efficiently, different value per row:** `bulk_update()`.
- **Creating efficiently, many rows at once:** `bulk_create()`.
- **Protecting against concurrent interference:**
  - Simple math on a field → `F()`.
  - Logic that depends on reading the current value before deciding what to do → `select_for_update()` inside `transaction.atomic()`.
