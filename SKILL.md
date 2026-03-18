---
name: django-best-practices
description: >
  Mandatory Django and Django REST Framework best practices guide. You MUST use this skill whenever writing,
  reviewing, optimizing, or refactoring Django projects, ORM queries, APIs, Celery tasks, WebSockets/Channels, or testing suites.
  It is critical for solving N+1 queries, optimizing database models, structuring architecture (Fat Models,
  Services layer), security hardening, DRF configurations, asynchronous views blocking the ORM, and Channels.
  Trigger IMMEDIATELY on phrases like "Django ORM query", "N+1 bugs", "queryset optimization", "DRF APIs",
  "serializers", "Celery tasks", "async Django", "Django Channels", "Django project structure", "Django testing", or "Django caching".
  DO NOT write Django or DRF code without consulting these rules if the user asks for high-quality,
  performant, or production-ready backend code.
---

# Django Best Practices

A stringent, high-fidelity guide to writing correct, highly performant Django and DRF — distilled from
real-world patterns, HackSoftware Styleguide, and advanced architectural pitfalls. Contains optimized rules across 15
critical categories, meticulously designed to surpass ordinary knowledge.

## When to Apply

You MUST reference these guidelines when:
- Writing new Django models, QuerySets, views, or endpoints
- Debugging slow SQL, ORM blocking, N+1 queries, or MemoryErrors
- Reviewing DRF serializers, complex aggregations, or bulk inserts
- Implementing architecture (fat models vs thin views, services layer, selectors)
- Deciding whether to use signals, Celery, or caching
- Writing test suites specifically for Django
- Working with WebSockets and Django Channels
- Writing migration logic or customizing the Django Admin panel natively.

---

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|---|---|---|---|
| 1 | Database & Queries | CRITICAL | `orm-` |
| 2 | Security & Hardening | CRITICAL | `security-` |
| 3 | Application Architecture | CRITICAL | `arch-` / `model-` |
| 4 | Advanced ORM Features | HIGH | `orm-` |
| 5 | Views & DRF APIs | HIGH | `views-` / `api-` |
| 6 | Tasks & Asynchronous | HIGH | `async-` |
| 7 | Django Channels | HIGH | `channels-` |
| 8 | Migrations & Admin | HIGH | `migrations-` / `admin-` |
| 9 | Database Locks | HIGH | `orm-` |
| 10 | Structured Logging | MEDIUM | `logging-` |
| 11 | Caching Strategies | MEDIUM | `caching-` |
| 12 | Implicit Signals | MEDIUM | `signals-` |
| 13 | Testing & Assertions | MEDIUM | `test-` |

---

## Quick Reference

### 1. Database & Queries (CRITICAL)
- `orm-nplusone` — Use `select_related()` and `prefetch_related()` explicitly to avoid executing loops of queries.
- `orm-only-defer` — Load strictly the exact columns needed into Python memory to save critical RAM.
- `orm-laziness` — Querysets evaluate lazily; understand when they execute to prevent duplicate queries.

### 2. Advanced ORM Features & Locks (HIGH)
- `orm-select-for-update` — Atomically lock rows fundamentally during financial transactions or queue polling.
- `orm-annotate` — Aggregate massive datasets completely within the database.
- `orm-f-expressions` — Cleanly prevent lost update bugs by pushing mathematical operations to the database explicitly.
- `orm-bulk-inserts` — Execute massive inserts in one optimal network request with `bulk_create`.

### 3. Application Architecture (CRITICAL)
- `arch-services-layer` — Write business logic in a DRY services layer, entirely removed from HTTP views.
- `model-fat-models` — Build complex reusable behaviors fundamentally inside model instances.
- `arch-selectors` — Encapsulate exact fetching operations cleanly in selector methods.

### 4. Views & DRF APIs (HIGH)
- `views-fbv-vs-cbv` — Use CBVs for routine generic endpoints, FBVs for distinct explicit custom functionality.
- `api-drf-serializers` — Validate logic totally explicitly in the serializer entirely separate from view context.
- `api-viewset-optimization` — Pre-optimize viewset querysets effortlessly via `.prefetch_related()`.

### 5. Tasks & Asynchronous (HIGH)
- `async-celery-idempotency` — Tasks must run securely multiple times without duplicating effects.
- `async-django-views` — Ensure ORM functions are completely wrapped in `sync_to_async` for async views natively.

### 6. Django Channels (HIGH)
- `channels-sync-to-async` — Wrap synchronous ORM queries inside async websocket consumers.
- `channels-channel-layers` — Send strictly JSON native primitives through channels layers.
- `channels-security-auth` — Properly authenticate ASGI Websocket connections natively.

### 7. Security (CRITICAL)
- `security-uuids-in-urls` — Replace sequential logic IDs fundamentally with UUIDs to avoid enumeration.
- `security-env-secrets` — Extract `SECRET_KEY` correctly into environment variables.
- `security-host-validation` — Do not use wildcard domains in `ALLOWED_HOSTS` to prevent Host poisoning.
- `security-csrf` — Never exempt CSRF tokens indiscriminately. Use header passing via AJAX explicitly.
- `security-xss` — Rely exclusively on Django auto-escaping and strictly sanitize raw HTML.

### 8. Migrations, Admin & Logging
- `admin-nplusone` / `admin-foreignkey-dropdowns` — Optimize Django admin on large tables explicitly via autocomplete.
- `migrations-runpython` — Keep data backfills isolated rigidly from schema edits.
- `logging-structured` — Completely replace `print()` with Python's strictly named `logging` natively.

---

## How to Use

Read individual rule files for detailed explanations and code examples:

```
rules/orm-queries.md
rules/security.md
rules/architecture.md
rules/database-locks.md
rules/admin.md
rules/migrations.md
```

Each exceptionally detailed rule file contains:
- **Why it matters** — The profound architectural truth it solves
- **❌ Wrong** — The catastrophic anti-pattern completely 
- **✅ Right** — The robust, optimized Django resolution 
- **Notes** — Connected architectural principles and caveats

## Full Compiled Document

For the comprehensive monolithic guide natively containing all expanded rules logically: `AGENTS.md`
