# Django ORM Notes — Custom Managers & Custom QuerySets (Detailed)

---

## 1. What is a Manager, Really?

Every Django model automatically gets a default manager called `objects`, even if you never write it yourself:

```python
class Product(models.Model):
    name = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=10, decimal_places=2)
```

Behind the scenes, Django attaches a `Manager` instance to this class, accessible as `Product.objects`. You didn't define it — Django adds it automatically because every model needs *some* way to query its table.

```python
Product.objects.all()
Product.objects.filter(price__gt=100)
Product.objects.create(name='Mouse')
```

**Core idea:** a Manager is the *gateway* between your model class and the database table as a whole. It's not about any single row — it's about the table.

---

## 2. Manager vs. Instance — Two Different Levels of Operation

This is a distinction worth internalizing clearly, because it explains *why* certain methods live in certain places:

| | Manager (`Product.objects`) | Instance (`product = Product(...)`) |
|---|---|---|
| Operates on | The **whole table** | **One single row** |
| Typical methods | `.all()`, `.filter()`, `.create()`, `.get()` | `.save()`, `.delete()` |
| Accessed via | The **class** (`Product.objects`) | An **object** (`product.save()`) |

**Why this split exists:** "give me all products under $50" is a table-wide question — it makes no sense to ask a single `Product` instance that. Conversely, "save yourself" or "delete yourself" only makes sense for one specific row — it makes no sense to ask the whole table to "save" as if it were one row.

So: **Manager = table-level actions. Instance = row-level actions.** Keeping this separation clean is exactly why Django designed it this way.

---

## 3. Custom Managers — Adding Your Own Table-Level Methods

Sometimes `.filter()`, `.all()`, etc. aren't expressive enough for your business logic. You end up repeating the same filter conditions everywhere in your codebase:

```python
# Repeated all over the codebase — error-prone, hard to maintain
Expense.objects.filter(is_paid=False)
Expense.objects.filter(is_paid=False)
Expense.objects.filter(is_paid=False)
```

If the definition of "unpaid" ever changes (e.g., you later add a `is_cancelled` field that should also count), you'd have to hunt down and update every single place this filter appears.

### Solution: define your own manager

```python
class ExpenseManager(models.Manager):
    def unpaid(self):
        return self.filter(is_paid=False)
```

Then attach it to the model (typically by assigning it as the model's manager, e.g. `objects = ExpenseManager()` inside the `Expense` model class).

### Usage

```python
Expense.objects.unpaid()
```

Now, "what counts as unpaid" is defined **in exactly one place**. If the business rule changes later, you edit `unpaid()` once, and every part of the codebase using it automatically gets the updated logic.

### Benefits

- **Reusable** — write the logic once, call it from anywhere.
- **Readable** — `Expense.objects.unpaid()` reads like plain English, communicating *intent* rather than raw filter conditions.
- **Centralized business logic** — the "meaning" of business concepts like "unpaid," "overdue," "active," etc. lives in one authoritative place instead of being scattered and possibly inconsistent across the codebase.

---

## 4. Why Manager Methods Should Return a QuerySet

Notice that `unpaid()` doesn't return a plain Python list — it returns `self.filter(...)`, which is a **QuerySet**.

```python
def unpaid(self):
    return self.filter(is_paid=False)   # returns a QuerySet, not a list
```

### Why this matters

QuerySets are **chainable** and **lazy** (see Day 1 notes). If your custom method returns a QuerySet instead of, say, a list of objects, the caller can keep building on top of it:

```python
Expense.objects.unpaid().count()
Expense.objects.unpaid().exists()
Expense.objects.unpaid().order_by('-amount')
```

If `unpaid()` instead did something like `return list(self.filter(is_paid=False))`, none of this chaining would work — a plain Python list doesn't understand `.count()`, `.exists()`, or `.order_by()` the way a QuerySet does, and the query would have already executed early, defeating the purpose of lazy evaluation.

**Rule of thumb:** unless you have a specific reason to force evaluation, custom manager/queryset methods should return QuerySets so the caller retains full flexibility.

---

## 5. `self` Inside a Manager

```python
class ExpenseManager(models.Manager):
    def unpaid(self):
        return self.filter(is_paid=False)
```

Here, `self` refers to the **current Manager instance itself** — i.e., `Expense.objects`. Calling `self.filter(...)` is exactly equivalent to calling `Expense.objects.filter(...)` directly — you're just doing it *from inside* the manager class definition, so you refer to the manager as `self` rather than by name.

---

## 6. The Big Limitation of Custom Managers

Here's the catch that trips a lot of people up:

```python
class ExpenseManager(models.Manager):
    def unpaid(self):
        return self.filter(is_paid=False)

    def expensive(self):
        return self.filter(amount__gt=1000)
```

You might expect this to work:

```python
Expense.objects.unpaid().expensive()   # ERROR!
```

**Why does this fail?**

- `Expense.objects` is a **Manager**.
- Calling `.unpaid()` on it returns a **QuerySet** (as established above), *not* another Manager.
- `.expensive()` is defined on the **Manager** class (`ExpenseManager`), not on a plain **QuerySet**.
- So once you've called `.unpaid()`, you're now holding a regular QuerySet object — and a plain QuerySet has no idea what `.expensive()` even means, because `expensive()` was only ever attached to the Manager, not to QuerySets in general.

**In short:** custom manager methods can call each other's underlying logic (e.g., by calling `self.filter(...)` repeatedly inside the manager), but you **cannot chain custom manager methods directly onto each other** the way you can chain built-in QuerySet methods — because after the first custom call, you're no longer working with a Manager at all, just a plain QuerySet.

---

## 7. The Fix: Custom QuerySets

The solution is to define the custom methods on a **QuerySet subclass** instead of directly on the Manager:

```python
class ExpenseQuerySet(models.QuerySet):
    def unpaid(self):
        return self.filter(is_paid=False)

    def expensive(self):
        return self.filter(amount__gt=1000)
```

Because both `unpaid()` and `expensive()` are now defined **on the QuerySet class itself**, calling one of them returns *another* `ExpenseQuerySet` — which *still has access to* `.expensive()`, `.unpaid()`, and every standard QuerySet method (`.filter()`, `.count()`, `.order_by()`, etc.).

### Result — full chaining now works

```python
Expense.objects.unpaid().expensive()
```

This works because:
1. `Expense.objects` starts as the manager.
2. `.unpaid()` returns an `ExpenseQuerySet` (not a plain generic QuerySet, and not a Manager).
3. That returned `ExpenseQuerySet` **still has** `.expensive()` available on it, since it's the same custom QuerySet class.
4. `.expensive()` further filters and returns yet another `ExpenseQuerySet`, so the chain could keep going indefinitely.

### Connecting a custom QuerySet to the Manager

To make `Expense.objects` actually *use* this custom QuerySet (so `Expense.objects.unpaid()` works as the entry point too), you typically hook it up via `Manager.from_queryset(...)` or by overriding `get_queryset()` in a manager — the key underlying idea is: **the manager's job becomes returning an instance of your custom QuerySet class**, rather than a plain default one, so all your custom methods are available right from `objects` onward and remain chainable indefinitely.

---

## 8. Custom Manager vs. Custom QuerySet — The Core Distinction

| | Custom Manager | Custom QuerySet |
|---|---|---|
| Defined by subclassing | `models.Manager` | `models.QuerySet` |
| Method return type | Plain **QuerySet** (loses the custom class) | The **same custom QuerySet class** |
| Chainable with other custom methods? | **No** — after one custom call, you're on a plain QuerySet | **Yes** — every custom method returns the same enriched class, so chaining continues |
| Best used for | A single, standalone reusable query, or as the *entry point* wrapping a custom QuerySet | Multiple related reusable filters meant to be **combined together** |

**Practical guidance:** if you only ever need one custom query in isolation (`Expense.objects.unpaid()` used alone, never combined with another custom method), a plain custom Manager method is fine. But the moment you want to **combine multiple custom filters together** (`unpaid().expensive().recent()`, etc.), you need a **custom QuerySet**, because only QuerySet methods preserve chainability with each other.

---

## Interview Summary — Quick Reference

| Concept | Meaning |
|---|---|
| **Manager** | The entry point to the database for a model — represents table-wide operations (`objects`). |
| **QuerySet** | A lazy, chainable representation of a database query — not the data itself, and not yet executed. |
| **Custom Manager** | Lets you define **reusable, named table-level queries** (e.g., `unpaid()`) so business logic isn't duplicated across the codebase. |
| **Custom QuerySet** | Lets you define **reusable queries that remain chainable with each other**, because each custom method returns the same enriched QuerySet class rather than a plain one. |

**One-line mental model:** *A Manager gets you into the database. A QuerySet is what lets you keep building your query once you're inside — and only a custom QuerySet lets your own custom filters keep chaining onto each other the way built-in ones do.*
