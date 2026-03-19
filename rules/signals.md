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
