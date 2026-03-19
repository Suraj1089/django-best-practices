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
