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
