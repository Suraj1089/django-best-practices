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
