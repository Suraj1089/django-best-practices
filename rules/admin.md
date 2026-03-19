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
