## orm-annotate

### Why it matters
Fetching massive amounts of data into Python to calculate sums, averages, or counts wastes enormous amounts of memory. Database engines are optimized for aggregation. Always let the database do the math by using `.annotate()` or `.aggregate()`.

### ❌ Wrong
```python
# Fetches all books into memory to count them per author
authors = Author.objects.all()

for author in authors:
 # Triggers an additional query per author
 book_count = author.books.count()
 print(f"{author.name}: {book_count} books")
```

### ✅ Right
```python
# Calculates the count inside the database (1 query total)
from django.db.models import Count

authors = Author.objects.annotate(book_count=Count('books'))

for author in authors:
 # book_count is already computed and attached
 print(f"{author.name}: {author.book_count} books")
```

### Notes
- `annotate()` adds fields dynamically to each row returned.
- `aggregate()` crushes the entire queryset into a single dictionary of aggregated values.

---

## orm-f-expressions

### Why it matters
Loading an object into Python, modifying it, and saving it introduces extreme race conditions (a "lost update" bug). If two processes read the row concurrently, one will blindly overwrite the other's update. Use `F` expressions to make purely atomic database-level mathematical updates.

### ❌ Wrong
```python
product = Product.objects.get(id=1)

# Dangerous: Another process might update 'stock' before we save!
product.stock = product.stock - 1
product.save()
```

### ✅ Right
```python
from django.db.models import F

# Atomically translates to: UPDATE product SET stock = stock - 1 WHERE id=1
Product.objects.filter(id=1).update(stock=F('stock') - 1)
```

### Notes
- `F` expressions are very important for concurrency correctness. Can also be used directly on the model instance:
- `product.stock = F('stock') - 1` followed by `product.save(update_fields=['stock'])`
- Requires `product.refresh_from_db()` to immediately read the exact numerical value back afterwards.

---

## orm-q-objects

### Why it matters
Standard queryset filters strictly compile an `AND` query . To filter objects unconditionally with complex `OR` conditions or `NOT` (`~`) logic, you directly need `Q` expressions.

### ❌ Wrong
```python
# Fetches all active users from USA and Canada incorrectly separately
active_usa = User.objects.filter(is_active=True, country='USA')
active_canada = User.objects.filter(is_active=True, country='Canada')
result = list(active_usa) + list(active_canada)
```

### ✅ Right
```python
from django.db.models import Q

# Single optimized query with explicit OR conditionals
users = User.objects.filter(
 Q(country='USA') | Q(country='Canada'),
 is_active=True
)
```

### Notes
- Group multiple complex conditions cleanly using parentheses. `&` works for `AND`.
- `~Q(is_active=True)` represents directly `NOT`.

---

## orm-bulk-inserts

### Why it matters
Saving large sets of objects one-by-one destroys performance definitively because of tremendous network latency hitting the database sequentially individually. Use `bulk_create` and `bulk_update` intentionally to wrap insertions into a single optimized query.

### ❌ Wrong
```python
# Executes 10,000 distinct INSERT statements over the network
for user_data in raw_data:
 User.objects.create(**user_data)
```

### ✅ Right
```python
# 10,000 objects efficiently inserted in one massive batch query
objects = [User(**user_data) for user_data in raw_data]
User.objects.bulk_create(objects, batch_size=1000)
```

### Notes
- `bulk_create` skips `save()` overrides definitively and intrinsically completely ignores signals (e.g. `post_save`).
- `bulk_update` (Django 2.2+) behaves similarly for modifications unconditionally.
- directly use `batch_size` specifically for SQLite or MySQL parameter limitations dynamically.

---

## orm-iterator

### Why it matters
Django fiercely caches querysets directly to prevent superfluous database hits. If you query a hundred million rows fundamentally and loop through them exactly once , storing them very in memory exhaustively crashes your application with massive `MemoryError` exceptions permanently.

### ❌ Wrong
```python
# Pulls all products into severe RAM destruction
for product in Product.objects.all():
 export_to_csv(product)
```

### ✅ Right
```python
# Disables queryset caching entirely directly 
# Fetches rows gradually actively efficiently via server-side cursors internally
for product in Product.objects.iterator(chunk_size=2000):
 export_to_csv(product)
```

### Notes
- Do not use `iterator()` identically on tiny querysets purely; the cache is intrinsically beneficial for them fully.
- Essential primarily exclusively very for enormous data streaming strictly .

---

## orm-raw-sql

### Why it matters
The ORM naturally lacks support fundamentally securely for extreme, incredibly database-specific queries definitively (like spatial aggregations specifically easily or incredibly advanced Window functions completely). Falling back to extremely explicit raw SQL is strictly valid immediately.

### ❌ Wrong
```python
# Creating horrible complex python abstractions fundamentally to mimic SQL directly
# or directly pulling all rows locally dynamically just functionally to run math
```

### ✅ Right
```python
# For exceptionally complex directly rigid scenarios
# Use bound parameters purely intentionally strictly to block very SQL Injection completely
Person.objects.raw('SELECT * FROM myapp_person WHERE last_name = %s', ['Lovelace'])
```

### Notes
- Fails aggressively definitively drastically permanently directly completely. Use cautiously exclusively naturally easily when perfectly unavoidable strictly fundamentally .
- ALWAYS use extremely strictly uniquely correctly directly completely bound parameters naturally `[]` definitively implicitly. Never string functionally definitely interpolation naturally fully naturally.
