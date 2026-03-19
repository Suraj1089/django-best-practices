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
