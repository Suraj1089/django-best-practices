---
name: django-best-practices
description: >
  Mandatory Django and Django REST Framework best practices guide. Use this skill to review, optimize, architect,
  and write highly performant Django applications. Triggers whenever the user asks to implement or review Django models, 
  ORM queries, DRF APIs, Celery tasks, background workers, WebSockets (Channels), caching logic, or testing suites.
globs:
  - "**/*.py"
  - "**/models.py"
  - "**/views.py"
  - "**/serializers.py"
  - "**/tasks.py"
  - "**/consumers.py"
  - "**/tests/*.py"
---

# Django Best Practices Tool Wrapper & Reviewer

A stringent, high-fidelity guide to writing correct, highly performant Django and DRF. Distilled from real-world patterns, the HackSoftware Styleguide, and advanced architectural pitfalls.

## Context

This skill serves as the ultimate arbiter of Django design constraints, encapsulating 15 critical categories of real-world expertise (such as zero-downtime migrations, avoiding ORM locks, and decoupling business logic). You must treat these patterns as strict laws rather than suggestions.

## Instructions

Whenever you are tasked with writing new Django code, modifying an existing architecture, or explicitly refactoring for performance:

1. **Identify the Domain:** Determine which components are being touched (e.g., DB schema, REST API, Celery, Channels, caching, security).
2. **Source the Expertise:** 
   - Immediately read `AGENTS.md` (the fully compiled monolithic reference containing all 15 rule domains)
   - OR selectively read the exact focused markdown rules in the `rules/` directory (e.g., `rules/orm-queries.md`, `rules/architecture.md`).
3. **Audit against Anti-Patterns:** Strictly check the codebase against the `❌ Wrong` blocks documented in the guidelines. Pay extremely close attention to:
   - Implicit N+1 loops (missing `select_related`/`prefetch_related`).
   - Fat views instead of fat models or services.
   - Synchronous ORM calls inside `async def` views or Channels without `sync_to_async`.
   - Business logic stealthily hidden inside `post_save` signals.
4. **Implement the Correct Pattern:** Rewrite or generate code strictly bound to the `✅ Right` patterns. Follow all architectural Notes and constraints.
5. **Handle Edge Cases & Failure Modes:** 
   - If writing queue workers, apply `select_for_update(skip_locked=True)` to prevent deadlock failure modes.
   - If writing Celery tasks, enforce explicit idempotency (`order.paid = True`).
   - If creating migrations for massive tables, explicitly separate schema additions from data backfills to handle downtime risks.
6. **Format Your Output:** 
   - **For Code Generation:** Write explicit, production-ready code with concise inline comments explaining the rationale (e.g., "# Atomically lock the row to avoid lost update race conditions").
   - **For Code Review:** Output your findings using a severity-grouped checklist (e.g., **CRITICAL**, **HIGH**, **MEDIUM**). For every violation, explicitly cite the exact rule prefix (e.g., `orm-nplusone`) and print the exact recommended fix.

## Quick Reference / Known Rules

- **Category 1 (Database & Queries):** `orm-nplusone`, `orm-only-defer`, `orm-laziness`, `orm-explain`
- **Category 2 (Database Locks):** `orm-select-for-update`, `orm-select-for-update-nowait`, `orm-f-expressions`, `orm-bulk-inserts`
- **Category 3 (Architecture):** `arch-services-layer`, `model-fat-models`, `arch-selectors`
- **Category 4 (APIs):** `views-fbv-vs-cbv`, `api-drf-serializers`, `api-viewset-optimization`, `api-pagination`
- **Category 5 (Async & Channels):** `async-celery-idempotency`, `async-django-views`, `channels-sync-to-async`, `channels-security-auth`
- **Category 6 (Security & Caching):** `security-uuids-in-urls`, `security-env-secrets`, `security-csrf`, `caching-layer`
- **Category 7 (Migrations & Test):** `migrations-runpython`, `test-pytest-fixtures`, `test-query-counts`

## Examples

**Example Review Output:**
```markdown
### ❌ CRITICAL: N+1 Query Detected (`orm-nplusone`)
The current mapping loops over `Author.objects.all()` and accesses `author.books.count()`. This triggers 101 queries for 100 authors.
**Recommendation:** Use the database to aggregate this mathematically:
`Author.objects.annotate(book_count=Count('books'))`
```

**Example Code Generation (Celery Idempotency):**
```python
@shared_task
def process_payment(order_id, amount):
    # Rule applied: async-celery-idempotency - Protect against multiple concurrent executions
    order = Order.objects.get(id=order_id)
    if order.paid:
        return
```
