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
