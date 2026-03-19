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
