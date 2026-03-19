## orm-nplusone

### Why it matters
The N+1 query problem occurs when you access a related object in a loop. Django's ORM is lazy, and accessing related data without directly fetching it beforehand causes the ORM to execute a separate SQL query for each item in the list, completely destroying performance for large datasets.

### ❌ Wrong
```python
# Fetches all books (1 query)
books = Book.objects.all()

for book in books:
 # Triggers a NEW query for each book to get its author
 # If there are 100 books, this executes 101 queries
 print(book.author.name)
```

### ✅ Right
```python
# Fetches all books AND their authors in a single JOIN (1 query total)
books = Book.objects.select_related('author').all()

for book in books:
 # Author data is already in memory, no new queries are triggered
 print(book.author.name)
```

### Notes
- Use `select_related()` for foreign key and one-to-one relationships (creates a SQL JOIN).
- Use `prefetch_related()` for many-to-many and reverse foreign key relationships (creates 2 queries and joins them in Python).

---

## orm-only-defer

### Why it matters
Fetching columns from the database that you don't actually use wastes database memory, network bandwidth, and application memory. If an object has large text blocks or JSON fields, loading it all heavily degrades optimization.

### ❌ Wrong
```python
# Fetches ALL columns from the user table
users = User.objects.all()

for user in users:
 # We only care about the username and email, but we fetched the entire profile/bio
 send_email(user.username, user.email)
```

### ✅ Right
```python
# Fetches ONLY the id, username, and email fields
users = User.objects.only('id', 'username', 'email')

for user in users:
 send_email(user.username, user.email)
```

### Notes
- `only()` restricts columns to what is specified (plus the primary key).
- `defer()` is the opposite; it fetches all columns EXCEPT the ones specified. 
- Warning: Accessing a deferred field later will trigger an immediate additional database query per object.

---

## orm-laziness

### Why it matters
Django QuerySets are lazy. They are not evaluated (sent to the database) until you actually iterate over them, slice them with a step, or call methods like `list()`, `len()`, or `bool()`. Not understanding laziness leads to evaluating querysets multiple times unnecessarily.

### ❌ Wrong
```python
users = User.objects.filter(is_active=True)

# Evaluates the queryset and fetches all objects to count them
if len(users) > 0:
 # Evaluates the queryset a SECOND time
 for user in users:
 print(user.name)
```

### ✅ Right
```python
users = User.objects.filter(is_active=True)

# Use .exists() if you just need to check if there are any results without loading them
# OR if you need the data, evaluate it once:
users_list = list(users) # Evaluates once
if users_list:
 for user in users_list:
 print(user.name)
```

### Notes
- Use `.exists()` to check for presence efficiently without retrieving rows.
- Use `.count()` to reliably get the count in SQL instead of fetching objects into memory.

---

## orm-explain

### Why it matters
It's notoriously difficult to guess exactly what SQL the Django ORM is generating and what indices the database considers. Not using `EXPLAIN ANALYZE` means you're flying blind regarding query performance.

### ❌ Wrong
```python
# Guessing that this is fast without profiling it
expensive_query = MyModel.objects.filter(name__icontains="test").order_by('-created_at')
results = list(expensive_query)
```

### ✅ Right
```python
# Print the exact query execution plan
expensive_query = MyModel.objects.filter(name__icontains="test").order_by('-created_at')

# Prints the database's query plan, showing sequential scans or index usage
print(expensive_query.explain(analyze=True))
```

### Notes
- `explain()` is available in Django 2.1+.
- `analyze=True` actually executes the query to return true execution times (PostgreSQL/MySQL support this).
