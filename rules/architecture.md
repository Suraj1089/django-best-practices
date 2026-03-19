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
