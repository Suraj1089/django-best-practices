# Django Best Practices — Full Reference
> Compiled from all rule files. For focused context, read rules/<rule>.md directly.
---
## admin-nplusone

### Why it matters
The Django Admin interface is often susceptible to severe N+1 queries. Displaying foreign keys inside `list_display` without selecting them triggers a separate database hit for every single row shown on the page.

### ❌ Wrong
```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
 # Customer is a ForeignKey. Rendering this page sends 101 queries for 100 rows.
 list_display = ('id', 'customer', 'total_amount')
```

### ✅ Right
```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
 list_display = ('id', 'customer', 'total_amount')
 
 # Pre-select related models effortlessly to optimize rendering down to 1 query
 list_select_related = ('customer',)
```

### Notes
- `list_select_related = True` automatically pre-fetches all `ForeignKey` relations present on the model, but it is directly preferable to pass a tuple targeting the exact tables needed.

---

## admin-foreignkey-dropdowns

### Why it matters
The default Django Admin renders every single ForeignKey as an HTML `<select>` dropdown populated with all possible foreign values. If you possess a `User` table with 2 million users, navigating to the admin view for an `Order` will blindly load 2 million users from the database into your web server's RAM in order to construct a gigantic `<option>` list crashing out of memory.

### ❌ Wrong
```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
 # Relies on the default behavior that loads the entire linked table into a dropdown
 pass
```

### ✅ Right
```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
 # Defends memory entirely by forcing the use of an directly minimal text field
 raw_id_fields = ('customer',)
 
# Alternatively, enforce a powerful searchable autocomplete widget cleanly:
@admin.register(User)
class UserAdmin(admin.ModelAdmin):
 search_fields = ['email']

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
 # Requires defining search_fields on the target UserAdmin securely
 autocomplete_fields = ('customer',)
```

### Notes
- Standardize completely on `autocomplete_fields` unconditionally for any model containing over a few thousand rows. 
- Ensure `search_fields` are adequately indexed cleanly .


---

## arch-services-layer

### Why it matters
Putting logic in views or serializers violates DRY. The services layer creates a single, reusable function for any meaningful business transaction, decoupled from HTTP endpoints.

### ❌ Wrong
```python
# View has massive business logic
def register_user(request):
 # check email, hash password, create user, send email, setup stripe
```

### ✅ Right
```python
# services/user_registration.py
def register_user(email, password):
 # Transaction atomic block
 with transaction.atomic():
 user = User.objects.create_user(email=email, password=password)
 UserProfile.objects.create(user=user)
 stripe_customer = stripe.Customer.create(email=email)
 user.stripe_id = stripe_customer.id
 user.save()
 return user

# views.py
def register_user_view(request):
 user = register_user(request.POST['email'], request.POST['password'])
```

### Notes
- Services are strict Python functions (not classes), take primary types/models as arguments, and do the heavy transaction management.

---

## arch-selectors

### Why it matters
Repeating complex `.filter().select_related().annotate()` querysets across your codebase leads to severe fragmentation. Selector functions encapsulate complex database reads to make them reusable anywhere.

### ❌ Wrong
```python
# In random places across the codebase
active_premium = User.objects.filter(is_active=True, plan='premium').select_related('profile')
```

### ✅ Right
```python
# selectors/users.py
def get_active_premium_users() -> QuerySet[User]:
 return User.objects.filter(
 is_active=True, 
 plan='premium'
 ).select_related('profile')
```

### Notes
- Selectors ONLY read data safely. Services ONLY write data comprehensively.


---

## async-celery-idempotency

### Why it matters
Celery tasks are guaranteed to run *at least once*, not *exactly once*. Network blips can cause a task to execute multiple times. If a task isn't idempotent (meaning running it 5 times has the exact same effect as running it 1 time), you will corrupt data.

### ❌ Wrong
```python
@shared_task
def process_payment(order_id, amount):
 # Dangerous: if this task retries, we charge the user multiple times!
 stripe.Charge.create(amount=amount)
 Order.objects.filter(id=order_id).update(paid=True)
```

### ✅ Right
```python
@shared_task
def process_payment(order_id, amount):
 order = Order.objects.get(id=order_id)
 # Idempotency lock: check if already processed
 if order.paid:
 return
 
 # Use an idempotency key with the external provider
 stripe.Charge.create(amount=amount, idempotency_key=f"order_{order.id}")
 order.paid = True
 order.save(update_fields=['paid'])
```

### Notes
- Always design tasks expecting them to blindly run twice simultaneously. Ensure database constraints or explicit locking handles conflicts cleanly.

---

## async-task-signatures

### Why it matters
Passing massive ORM objects into Celery task signatures breaks fundamentally when the task is picked up by a worker much later. The database state will have radically changed, and the serialized object will be stale.

### ❌ Wrong
```python
# Passing an entire ORM object into the message broker
@shared_task
def send_email(user):
 mail.send(user.email)

# caller: send_email.delay(user_object)
```

### ✅ Right
```python
# Pass ONLY the primary key integer
@shared_task
def send_email(user_id):
 # Fetch fresh data from the database inside the worker
 user = User.objects.get(id=user_id)
 mail.send(user.email)

# caller: send_email.delay(user.id)
```

### Notes
- Task arguments must be perfectly JSON serializable.

---

## async-django-views

### Why it matters
Django supports `async def` views since 3.1. However, making a view `async` completely destroys performance if you directly block it with synchronously slow Django ORM queries without wrapping them.

### ❌ Wrong
```python
async def my_view(request):
 # Blocks the async event loop permanently
 users = User.objects.all()
 return JsonResponse({'users': len(users)})
```

### ✅ Right
```python
from asgiref.sync import sync_to_async

async def my_view(request):
 # Wrap ORM calls correctly
 count = await sync_to_async(User.objects.count)()
 return JsonResponse({'users': count})
```

### Notes
- Always wrap ORM queries in `sync_to_async` inside an async view.


---

## caching-layer

### Why it matters
Caching the wrong layer causes cache stampedes and stale data bugs. Default to per-view caching carefully, and avoid caching user-specific HTML globally.

### ❌ Wrong
```python
# Caches a view globally that contains a user's private shopping cart
@cache_page(60 * 15)
def cart_view(request):
 return render(request, 'cart.html', {'cart': request.user.cart})
```

### ✅ Right
```python
# Cache directly by user or use template fragment caching.
@cache_page(60 * 15, key_prefix="public_home")
def public_view(request):
 return render(request, 'public.html')
```

### Notes
- `cache_page` caches the EXACT URL correctly. 
- Use template fragment caching `{% cache 500 sidebar request.user.username %}` for personalized static content.


---

## channels-sync-to-async

### Why it matters
Django's ORM is synchronous. If you perform any database queries directly inside an asynchronous WebSocket Consumer (e.g., `AsyncWebsocketConsumer`), you will block the single asynchronous event loop. This severe mistake severely throttles all active WebSocket connections simultaneously.

### ❌ Wrong
```python
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
 async def receive(self, text_data):
 # Disastrous: Synchronous ORM call directly blocking the ASGI loop
 user = User.objects.get(username="test")
 
 await self.send(text_data=f"Hello, {user.username}")
```

### ✅ Right
```python
from channels.generic.websocket import AsyncWebsocketConsumer
from channels.db import database_sync_to_async

class ChatConsumer(AsyncWebsocketConsumer):
 async def receive(self, text_data):
 # Await the ORM query cleanly inside a separate threadpool
 user = await self.get_user("test")
 
 await self.send(text_data=f"Hello, {user.username}")
 
 # Decorate synchronous logic directly 
 @database_sync_to_async
 def get_user(self, username):
 return User.objects.get(username=username)
```

### Notes
- Any interaction with Django models from an `async def` function requires `database_sync_to_async`.

---

## channels-channel-layers

### Why it matters
Channel layers facilitate cross-process communication (via Redis). Sending massive python ORM objects directly crashes the serializer. Only raw valid JSON primitives efficiently traverse over the Redis channel.

### ❌ Wrong
```python
class PostNotifier(AsyncWebsocketConsumer):
 async def notify_post_created(self, post_instance):
 # Fails : You cannot send native Django ORM models across a channel.
 await self.channel_layer.group_send(
 'post_notifications',
 {
 'type': 'new_post',
 'post': post_instance # Unserializable 
 }
 )
```

### ✅ Right
```python
class PostNotifier(AsyncWebsocketConsumer):
 async def notify_post_created(self, post_id, title):
 await self.channel_layer.group_send(
 'post_notifications',
 {
 'type': 'new_post',
 # JSON-serializable primitives
 'post_id': post_id,
 'title': title
 }
 )
```

### Notes
- Send only `str`, `int`, `float`, `dict`, and `list`. 

---

## channels-security-auth

### Why it matters
WebSockets bypass normal view-level permissions. You must directly authenticate the ASGI connection.

### ❌ Wrong
```python
# Directly accepting all connections without authentication 
class PrivateDataConsumer(AsyncWebsocketConsumer):
 async def connect(self):
 await self.accept() # Accepts anonymous users
```

### ✅ Right
```python
class PrivateDataConsumer(AsyncWebsocketConsumer):
 async def connect(self):
 if self.scope["user"].is_anonymous:
 # Drop the connection for unauthenticated users
 await self.close()
 else:
 await self.accept()
```

### Notes
- Ensure `AuthMiddlewareStack` is correctly configured in your `asgi.py`.


---

## orm-select-for-update

### Why it matters
When multiple processes (like concurrent API requests or Celery workers) attempt to read and modify the exact same database row simultaneously, one process will overwrite the other's changes. This severe bug is called a "race condition", resulting in lost money, oversold inventory, or duplicate charges.

### ❌ Wrong
```python
# Two concurrent transactions will both read balance=100.
# Both will add 50 and set balance=150.
# The final result is 150 instead of 200.
def deposit_money(wallet_id, amount):
 wallet = Wallet.objects.get(id=wallet_id)
 wallet.balance += amount
 wallet.save()
```

### ✅ Right
```python
from django.db import transaction

# directly lock the row using physical database-level row locks
@transaction.atomic
def deposit_money(wallet_id, amount):
 # This process locks the row. Any other concurrent request simply waits here
 # until the first transaction cleanly completes and releases the lock.
 wallet = Wallet.objects.select_for_update().get(id=wallet_id)
 wallet.balance += amount
 wallet.save(update_fields=['balance'])
```

### Notes
- `select_for_update()` must always be used strictly inside an explicit `@transaction.atomic` block to be active.
- For related objects, use `select_for_update(of=('self', 'related_model'))` to lock tables surgically.

---

## orm-select-for-update-nowait

### Why it matters
Occasionally, having a process wait for a lock is a terrible idea—for example, a polling queue worker checking if a job is free. By skipping already-locked rows implicitly, the application scales dramatically without creating database deadlocks.

### ❌ Wrong
```python
# A worker pulls jobs, but if another worker has the lock, it just hangs forever.
@transaction.atomic
def process_job():
 job = Job.objects.select_for_update().filter(status='pending').first()
 # Execute job
```

### ✅ Right
```python
@transaction.atomic
def process_job():
 # Use skip_locked=True. If another worker already locked the row, we simply grab the next one.
 job = Job.objects.select_for_update(skip_locked=True).filter(status='pending').first()
 if job:
 job.status = 'processing'
 job.save()
 # Execute job
```

### Notes
- `skip_locked=True` is invaluable for preventing worker contention on high-throughput queue systems residing inside Postgres or MySQL tables.


---

## logging-structured

### Why it matters
Calling `print()` dumps unsearchable unstructured text into stdout. When debugging scale apps across 10 servers, you need centralized searchable logs.

### ❌ Wrong
```python
def process_webhook(request):
 print("Received webhook request from Stripe") # Completely unsearchable in production
 print(f"User UUID: {request.user.uuid}")
 
 try:
 data = request.json()
 except Exception as e:
 # Fails silently in standard output buffers 
 print(e)
```

### ✅ Right
```python
import logging

# Always instantiate a named logger directly
logger = logging.getLogger(__name__)

def process_webhook(request):
 logger.info("Received Stripe webhook", extra={"user_uuid": str(request.user.uuid)})
 
 try:
 data = request.json()
 except Exception as e:
 # Pushes exact stacktraces smoothly to Datadog/Sentry 
 logger.exception("Failed to parse Stripe JSON payload", exc_info=e)
```

### Notes
- Configure `LOGGING` structurally in `settings.py` and map it to a JSON formatter.
- Rely on `logger.exception()` directly inside except blocks to capture the entire traceback perfectly directly safely.


---

## migrations-runpython

### Why it matters
Data migrations (`RunPython`) execute arbitrary Python logic over database rows. Mixing standard schema alterations (like adding a column) with data backfills directly within the exact same migration file prevents reverse rollbacks and can leave tables logically corrupted if the Python script crashes midway.

### ❌ Wrong
```python
# 0005_add_status_and_populate.py
class Migration(migrations.Migration):
 operations = [
 # Schema change
 migrations.AddField('Order', 'status', models.CharField(max_length=20)),
 # Mixed directly with custom Data iteration
 migrations.RunPython(populate_status)
 ]
```

### ✅ Right
```python
# 0005_add_status_column.py
class Migration(migrations.Migration):
 operations = [
 migrations.AddField('Order', 'status', models.CharField(max_length=20, null=True)),
 ]

# 0006_populate_status_data.py
# Creating a completely isolated file for the data logic 
class Migration(migrations.Migration):
 dependencies = [('orders', '0005_add_status_column')]
 operations = [
 migrations.RunPython(populate_status, reverse_code=migrations.RunPython.noop)
 ]
```

### Notes
- Use `python manage.py makemigrations --empty <myapp>` directly to create blank data migrations easily.
- Always provide a `reverse_code` directly so developers can effectively reverse branches functionally.

---

## migrations-default-values

### Why it matters
Adding a completely new column to a database table containing 5 million rows requires writing a default value to exactly 5 million rows simultaneously. This operation fully locks the table for writes in PostgreSQL (until Postgres 11) or MySQL .

### ❌ Wrong
```python
class Migration(migrations.Migration):
 operations = [
 # Adding a default to an existing massive table locks the table entirely 
 migrations.AddField('User', 'is_verified', models.BooleanField(default=False)),
 ]
```

### ✅ Right
```python
class Migration(migrations.Migration):
 operations = [
 # Step 1: Add the column as nullable so it executes instantaneously
 migrations.AddField('User', 'is_verified', models.BooleanField(null=True)),
 
 # Step 2: In a separate Data Migration, backfill records chunk by chunk
 
 # Step 3: Add the strict NOT NULL constraint backward-compatibly in a 3rd migration
 ]
```

### Notes
- Massive schema design alterations mandate splitting `AddField` from `AlterField` naturally to evade downtime creatively.


---

## model-fat-models

### Why it matters
Putting business logic in views makes it impossible to reuse that code elsewhere (management commands, Celery tasks, signals, other views). Pushing logic down into models ("Fat Models") or a separate service layer ensures the code is DRY and directly testable without mocking HTTP requests.

### ❌ Wrong
```python
def publish_post_view(request, post_id):
 post = get_object_or_404(Post, id=post_id)
 # Business logic incorrectly hidden inside the view
 if not post.is_published:
 post.is_published = True
 post.published_at = timezone.now()
 post.save()
 send_publish_email(post.author)
 return redirect('post_detail', post_id=post.id)
```

### ✅ Right
```python
# In the models.py
class Post(models.Model):
 def publish(self):
 if not self.is_published:
 self.is_published = True
 self.published_at = timezone.now()
 self.save(update_fields=['is_published', 'published_at'])
 send_publish_email(self.author)

# In the views.py
def publish_post_view(request, post_id):
 post = get_object_or_404(Post, id=post_id)
 # The view is "thin" – it only routes data and calls the model API
 post.publish()
 return redirect('post_detail', post_id=post.id)
```

### Notes
- Avoid completely overloading models with too many methods. For massive apps, extract complex workflows into a distinct `services.py` layer.

---

## model-managers-vs-querysets

### Why it matters
Writing complex `filter()` chains across your codebase heavily violates DRY. While Custom Managers encapsulate fetching data, a Custom Manager returning custom QuerySets is the best way to allow chaining methods together.

### ❌ Wrong
```python
class PostManager(models.Manager):
 def published(self):
 return self.filter(status='published')
 
# Error: You cannot chain these method calls 
posts = Post.objects.published().filter(author=user) 
```

### ✅ Right
```python
class PostQuerySet(models.QuerySet):
 def published(self):
 return self.filter(status='published')
 
 def high_rating(self):
 return self.filter(rating__gte=4)

# Attach the custom QuerySet fundamentally to a Manager
class Post(models.Model):
 objects = PostQuerySet.as_manager()

# You can strictly chain custom queryset filters
best_posts = Post.objects.published().high_rating()
```

### Notes
- Creating directly a custom `models.QuerySet` mapped to `.as_manager()` drastically simplifies query architecture.

---

## model-indexes

### Why it matters
Creating indexes strictly via `Meta.indexes` (instead of `db_index=True` or `index_together`) is the modern standard . `Index()` permits assigning explicit names (solving compatibility crashes), adding condition parameters for partial indexes, and performing zero-downtime lock-free additions via PostgreSQL's `AddIndexConcurrently`.

### ❌ Wrong
```python
# Discard deprecated index_together and limited db_index=True
class Book(models.Model):
 title = models.CharField(max_length=200, db_index=True)
 status = models.CharField(max_length=20)
 
 class Meta:
 index_together = [['status', 'title']]
```

### ✅ Right
```python
class Book(models.Model):
 title = models.CharField(max_length=200)
 status = models.CharField(max_length=20)

 class Meta:
 indexes = [
 # Explicit, modernized Index blocks completely replacing db_index
 models.Index(
 fields=['status', 'title'], 
 name='book_status_title_idx'
 ),
 models.Index(
 fields=['title'],
 name='book_title_idx'
 )
 ]
```

### Notes
- Consolidate all database indexing cleanly directly inside `Meta.indexes`.
- Name indexes securely directly securely. Do not rely on Django's auto-generated hashed names in order to write clean zero-downtime database upgrades later.

---

## model-constraints

### Why it matters
Validating cleanly strictly robustly at the Django application level fails during race conditions or bulk operations. Constraints pushed directly into the backend database enforce integrity perfectly.

### ❌ Wrong
```python
# Validation in Python heavily suffers from race conditions 
if not User.objects.filter(email=email).exists():
 User.objects.create(email=email)
```

### ✅ Right
```python
class User(models.Model):
 email = models.EmailField(unique=True) # Always rely on DB constraints

 class Meta:
 constraints = [
 models.UniqueConstraint(fields=['room', 'date'], name='unique_booking'),
 models.CheckConstraint(check=models.Q(age__gte=18), name='age_gte_18')
 ]
```

### Notes
- `UniqueConstraint` and `CheckConstraint` (Django 2.2+) push data validity to the database engine effectively.

---

## model-null-strings

### Why it matters
Using `null=True` on string-based fields means there are exactly two ways to represent "no data": `NULL` and the empty string `""`. This causes redundant checks everywhere in your code.

### ❌ Wrong
```python
# Bad: To check for no bio, you must check both None and ""
bio = models.TextField(null=True, blank=True)
```

### ✅ Right
```python
# Good: The only empty value is ""
bio = models.TextField(default="", blank=True)
```

### Notes
- Django convention: String-based fields should never have `null=True`.
- Exception: If the field has `unique=True`, use `null=True` to allow multiple empty entries safely.


---

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


---

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


---

## security-uuids-in-urls

### Why it matters
Exposing autoincrementing integer primary keys reveals business intelligence (how many users/orders you have) and enables IDOR (Insecure Direct Object Reference) and enumeration attacks if permissions are missing.

### ❌ Wrong
```python
# Exposes the exact database sequence ID
path('users/<int:user_id>/', user_profile, name='user_profile')

# Attacker can easily guess users/1/, users/2/, users/3/ and scrape your entire database
```

### ✅ Right
```python
import uuid
from django.db import models

class User(models.Model):
 # Create a secure, unguessable UUID
 uuid = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)

# URL completely prevents enumeration attacks
path('users/<uuid:user_uuid>/', user_profile, name='user_profile')
```

### Notes
- Integers are completely fine for internal foreign keys for performance reasons.
- Expose ONLY uniquely secure UUIDs or slugs to the public web interface.

---

## security-env-secrets

### Why it matters
Hardcoding `SECRET_KEY`, database credentials, or API keys directly in source code allows anyone with repository access to completely compromise your servers and impersonate any user via forged session cookies.

### ❌ Wrong
```python
# settings.py
DEBUG = True
SECRET_KEY = "django-insecure-hardcoded-secret"
DATABASES = {
 'default': {
 'USER': 'postgres',
 'PASSWORD': 'super_secret_password'
 }
}
```

### ✅ Right
```python
# settings.py
import environ

env = environ.Env()

# Fails and cleanly if the environment variable is fundamentally missing
DEBUG = env.bool('DJANGO_DEBUG', default=False)
SECRET_KEY = env('DJANGO_SECRET_KEY')

DATABASES = {
 'default': env.db('DATABASE_URL')
}
```

### Notes
- Use `django-environ` to easily parse types directly from the environment.
- NEVER set `DEBUG = True` in production. It exposes full tracebacks containing sensitive environmental variables directly in the browser.

---

## security-csrf

### Why it matters
Cross Site Request Forgery (CSRF) allows a malicious site to trick a user's browser into executing unwanted actions on another site where they are authenticated. Disabling CSRF protection blindly leaves your users vulnerable to full account takeover.

### ❌ Wrong
```python
from django.views.decorators.csrf import csrf_exempt

# Blindly exempting a view because the developer didn't want to figure out how to pass the token
@csrf_exempt
def update_profile(request):
 request.user.email = request.POST['email']
 request.user.save()
 return HttpResponse('Profile updated')
```

### ✅ Right
```python
# In templates, always use the CSRF token
<form method="post">
 {% csrf_token %}
 <input type="email" name="email">
 <button type="submit">Update</button>
</form>

# In AJAX requests, dynamically fetch the token from cookies and set the X-CSRFToken header
```

### Notes
- `csrf_exempt` should ONLY be used strictly for machine-to-machine API endpoints that directly rely on completely different authentication mechanisms (e.g., TokenAuthentication, JWT) instead of browser cookies.

---

## security-host-validation

### Why it matters
If `ALLOWED_HOSTS` is misconfigured or empty (or `*`), an attacker can craft requests with a fake `Host` header. This poisons password reset emails , causing password reset links to point directly to the attacker's server domain.

### ❌ Wrong
```python
# settings.py
ALLOWED_HOSTS = ['*'] # Completely disables Host header validation
```

### ✅ Right
```python
# settings.py
# Only directly strictly accept exact known domains
ALLOWED_HOSTS = ['www.myapp.com', 'myapp.com']
```

### Notes
- Always strictly validate the exact domain. For dynamic subdomains, use `'.myapp.com'` strategically.

---

## security-xss

### Why it matters
Cross-Site Scripting (XSS) lets attackers execute arbitrary JavaScript in victim browsers. Django protects against XSS by auto-escaping templates. If you bypass this without directly sanitizing user input, you introduce a critical vulnerability.

### ❌ Wrong
```html
<!-- Disabling auto-escaping blindly to render raw HTML from an untrusted user -->
<div class="user-bio">
 {{ user.bio|safe }} 
</div>
```

### ✅ Right
```html
<!-- Let Django auto-escape by default unconditionally -->
<div class="user-bio">
 {{ user.bio }}
</div>
```

### Notes
- If you absolutely must render HTML , strongly consider using `bleach` to thoroughly sanitize dangerous tags before using `|safe`.


---

## signals-explicit-calls

### Why it matters
Django signals are implicit, meaning the code that triggers them has no idea they exist. This causes profound debugging nightmares (spaghetti execution) and can randomly break transaction integrity. Prefer explicit service layer function calls instead.

### ❌ Wrong
```python
# models.py
@receiver(post_save, sender=User)
def send_welcome_email(sender, instance, created, **kwargs):
 if created:
 # Hidden side effect. Fails if called during a massive bulk_create!
 emailer.send(instance.email)
```

### ✅ Right
```python
# services.py
def create_user(email, password):
 user = User.objects.create(email=email, password=password)
 # directly visible side-effect. Easy to mock in tests.
 emailer.send(user.email)
 return user
```

### Notes
- ONLY use signals when decoupling two completely separate generic reusable apps (e.g., your custom app needs to hook into Django's built-in `auth` User model changes). 
- Never use signals for sequential business logic within the same app domain.

---

## signals-no-business-logic

### Why it matters
Putting core business logic inside signals fundamentally breaks predictability. If you need to update a related model, do it directly . Signals also heavily delay tests and slow down naive database operations unexpectedly.

### ❌ Wrong
```python
@receiver(post_save, sender=Order)
def update_inventory(sender, instance, **kwargs):
 # Core business logic hidden in a signal
 instance.product.stock -= instance.quantity
 instance.product.save()
```

### ✅ Right
```python
# Use a specific explicit method or service
class Order(models.Model):
 def finalize_checkout(self):
 self.status = 'COMPLETED'
 self.save()
 
 # Explicit update
 Product.objects.filter(id=self.product_id).update(
 stock=F('stock') - self.quantity
 )
```

### Notes
- Signals do not run during `bulk_create` or `bulk_update`. If your logic strictly lives in a signal, bulk operations will silently fail to execute it, leading to massive corrupted state.


---

## test-pytest-fixtures

### Why it matters
`pytest-django` combined with `factory_boy` cleanly outperforms standard fixtures by allowing dynamic, explicit, reusable object creation avoiding database integrity headaches entirely.

### ❌ Wrong
```python
# Using static JSON fixtures
class TestUser(TestCase):
 fixtures = ['users.json']
 
 def test_user(self):
 user = User.objects.get(id=1) # Hardcoded, brittle
```

### ✅ Right
```python
# Using factory_boy
class UserFactory(factory.django.DjangoModelFactory):
 class Meta:
 model = User
 username = factory.Faker('user_name')

@pytest.mark.django_db
def test_user_creation():
 user = UserFactory() # Automatically dynamic easily
 assert user.username is not None
```

### Notes
- Always fundamentally prefer generating data dynamically rather than loading static SQL/JSON dumps using `fixtures=`.

---

## test-query-counts

### Why it matters
The easiest way to accidentally cause an N+1 query bug is to change a Serializer. Always test query counts strictly using tools to assert query limits implicitly.

### ❌ Wrong
```python
@pytest.mark.django_db
def test_api_view(client):
 # No protection against N+1 regression
 response = client.get('/api/users/')
```

### ✅ Right
```python
@pytest.mark.django_db
def test_api_view(client, django_assert_num_queries):
 UserFactory.create_batch(10)
 
 with django_assert_num_queries(2): # 1 for count, 1 for users
 response = client.get('/api/users/')
```

### Notes
- Test query limits efficiently easily to catch performance bugs in CI.


---

## views-fbv-vs-cbv

### Why it matters
Standardizing on when to use Class-Based Views (CBVs) or Function-Based Views (FBVs) prevents unreadable views. CBVs excel at handling generic patterns (list, detail, create). FBVs excel at explicit complex procedural logic.

### ❌ Wrong
```python
# Re-inventing the wheel with an FBV for standard generic operations
def post_list(request):
 posts = Post.objects.all()
 paginator = Paginator(posts, 10)
 page_obj = paginator.get_page(request.GET.get('page'))
 return render(request, 'post_list.html', {'page_obj': page_obj})

# Using a CBV for a highly complex custom procedural task
class GenerateReportView(View):
 def get(self, request, *args, **kwargs):
 # 100 lines of complex procedural math and file generation
 # Better suited for an FBV
 pass
```

### ✅ Right
```python
# Use generic CBVs directly for CRUD 
class PostListView(ListView):
 model = Post
 paginate_by = 10
 template_name = 'post_list.html'

# Use FBVs directly for distinct, procedural customized actions
@require_POST
def generate_report(request):
 data = build_report_data()
 return JsonResponse(data)
```

### Notes
- Stick to `django.views.generic` exclusively for simple CRUD.
- Fall back strictly to FBVs for anything procedural.

---

## api-drf-serializers

### Why it matters
Putting custom data validation inside API views severely violates DRY protocols. DRF provides Serializers to securely validate data. Keep validation in serializers, not views.

### ❌ Wrong
```python
@api_view(['POST'])
def create_customer(request):
 # View contains validation logic
 if 'email' not in request.data:
 return Response({'error': 'Email required'}, status=400)
 if not '@' in request.data['email']:
 return Response({'error': 'Invalid email'}, status=400)
 
 Customer.objects.create(**request.data)
 return Response({'status': 'ok'})
```

### ✅ Right
```python
class CustomerSerializer(serializers.ModelSerializer):
 class Meta:
 model = Customer
 fields = ['email', 'name']

 def validate_email(self, value):
 if '@' not in value:
 raise serializers.ValidationError("Invalid email")
 return value

@api_view(['POST'])
def create_customer(request):
 serializer = CustomerSerializer(data=request.data)
 serializer.is_valid(raise_exception=True)
 serializer.save()
 return Response({'status': 'ok'})
```

### Notes
- Use `raise_exception=True` on `.is_valid()` to completely avoid `if` statements in views.

---

## api-viewset-optimization

### Why it matters
DRF ModelViewSets directly evaluate `.queryset` automatically. Returning a naive queryset triggers N+1 problems dramatically whenever a serializer tries to map foreign keys.

### ❌ Wrong
```python
class AuthorViewSet(viewsets.ModelViewSet):
 # Serializer automatically accesses author.books, triggering N+1 queries
 queryset = Author.objects.all()
 serializer_class = AuthorSerializer
```

### ✅ Right
```python
class AuthorViewSet(viewsets.ModelViewSet):
 # Optimize the queryset easily 
 queryset = Author.objects.prefetch_related('books').all()
 serializer_class = AuthorSerializer
```

### Notes
- Pre-fetch relations securely directly inside the `queryset` attribute.

---

## api-pagination

### Why it matters
Returning all items from a single model effectively overloads servers and network transmission during growth.

### ❌ Wrong
```python
class ProductViewSet(viewsets.ModelViewSet):
 queryset = Product.objects.all()
 serializer_class = ProductSerializer
 pagination_class = None # DANGEROUS!
```

### ✅ Right
```python
# In settings.py:
REST_FRAMEWORK = {
 'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
 'PAGE_SIZE': 50
}
```

### Notes
- NEVER return unpaginated lists.


---

