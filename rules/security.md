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
