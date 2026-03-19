---
name: django-best-practices
description: >
  Django and Django REST Framework best practices guide. Use this skill to review, optimize, architect,
  and write performant Django applications. Triggers when the user asks to implement or review Django models, 
  ORM queries, DRF APIs, Celery tasks, background workers, WebSockets (Channels), caching logic, or tests.
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

A strict guide to writing correct and performant Django and DRF code. Based on real-world patterns, the HackSoftware Styleguide, and common architectural pitfalls.

## Context

This skill is the final word on Django design constraints. It covers 15 core categories of real-world expertise (like zero-downtime migrations, avoiding ORM locks, and decoupling business logic). Treat these patterns as strict rules rather than suggestions.

## Instructions

Whenever you write new Django code, modify existing architecture, or refactor for performance:

1. **Identify the Domain:** Determine which components you are touching (e.g., DB schema, REST API, Celery, Channels, caching, security).
2. **Source the Expertise:** 
   - Read `AGENTS.md` (the compiled reference containing all 15 rule domains).
   - Or read the specific markdown rules in the `rules/` directory (e.g., `rules/orm-queries.md`, `rules/architecture.md`).
3. **Audit against Anti-Patterns:** Check the codebase against the `❌ Wrong` blocks documented in the guidelines. Look out for:
   - Implicit N+1 loops (missing `select_related`/`prefetch_related`).
   - Fat views instead of fat models or services.
   - Synchronous ORM calls inside `async def` views or Channels without `sync_to_async`.
   - Business logic hidden inside `post_save` signals.
4. **Implement the Correct Pattern:** Rewrite or generate code that follows the `✅ Right` patterns. Follow all architectural notes and constraints.
5. **Handle Edge Cases & Failure Modes:** 
   - When writing queue workers, use `select_for_update(skip_locked=True)` to prevent deadlocks.
   - When writing Celery tasks, enforce explicit idempotency (`order.paid = True`).
   - When creating migrations for large tables, separate schema additions from data backfills to handle downtime risks.
6. **Format Your Output:** 
   - **For Code Generation:** Write explicit, production-ready code with short inline comments explaining your reasoning (e.g., "# Atomically lock the row to avoid lost update race conditions").
   - **For Code Review:** Output your findings as a checklist grouped by severity (e.g., **CRITICAL**, **HIGH**, **MEDIUM**). For every violation, cite the exact rule prefix (e.g., `orm-nplusone`) and print the recommended fix.

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
