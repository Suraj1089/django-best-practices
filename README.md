# Django Best Practices Skill

An AI Agent skill that enforces best practices in Django, Django REST Framework, WebSockets (Channels), and asynchronous architectures.

This skill is built for integration into AI-agent workspace environments (like Claude Code, Cursor, Copilot, or Windsurf). When installed, it acts as a guide for avoiding architectural pitfalls, fixing N+1 queries, improving security, and optimizing database workloads.

---

## What it does

This skill helps your AI agent write better Django code. AI models often have outdated or contradictory knowledge about Django. This overrides generic suggestions with modern rules for building scalable apps.

It helps agents avoid:
- **Performance bottlenecks:** Unpaginated querysets, missing `select_related`/`prefetch_related`, loading millions of rows into RAM.
- **Architectural Mistakes:** Overloaded views, using plain classes when service layer functions are better, putting deep validation logic in generic views instead of DRF serializers.
- **Security Flaws:** Auto-incrementing primary IDs in web URLs (IDOR attacks), misconfigured CSRF/XSS strategies, hardcoding `SECRET_KEY` inside `settings.py`.
- **Concurrency issues:** Failing to use Database Row Locks (`select_for_update`) during queue polling or financial data mutations.

---

## Internal Structure

The core rules are broken down into logical files located in the `rules/` directory context, keeping token overhead low.

- `rules/admin.md`: Django admin optimizations (`autocomplete_fields`).
- `rules/architecture.md`: Services Layer, Data Selectors, DRY principles.
- `rules/async-and-tasks.md`: Handling Celery edge cases and async views.
- `rules/caching.md`: Guarding against cache stampedes.
- `rules/channels.md`: Safe Redis usage, ASGI auth.
- `rules/database-locks.md`: Stopping race conditions using Postgres/MySQL row locks.
- `rules/logging.md`: Enforcing standard JSON logging over `print()`.
- `rules/migrations.md`: Zero-downtime database changes.
- `rules/models.md`: Modernized `Meta.indexes`, database constraints, and Fat Models.
- `rules/orm-advanced.md`: Explicit database annotations and using `bulk_create`.
- `rules/orm-queries.md`: Fixing N+1 queries.
- `rules/security.md`: Hardening servers against IDOR and enumerations.
- `rules/signals.md`: Why you should avoid implicit signals for business logic.
- `rules/testing.md`: Enforcing PyTest over unittest, asserting query counts, and using factory_boy.
- `rules/views-and-apis.md`: Separating FBVs from CBVs and optimizing DRF endpoints.

---

## Usage & Setup

1. Clone this skill directory directly into your project's agent workspace.
2. If using an AI agent registry, install this via the CLI or UI to ensure `SKILL.md` is registered so system prompts know when to scan this knowledge base.
3. The centralized `AGENTS.md` file serves as a monolithic compilation designed for legacy AI interfaces requiring single-file text injections.

By following these constraints, agents generating Django code will write code that scales, avoids regressions, and passes security audits without requiring a human to manually point out common mistakes.
