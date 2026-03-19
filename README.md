# Django Best Practices Skill

A meticulously crafted, production-ready AI Agent skill designed to strictly enforce best practices in Django, Django REST Framework, WebSockets (Channels), and asynchronous architectures.

This skill is structured precisely for integration into [skills.sh](https://skills.sh) (or similar AI-agent workspace environments). When installed, it acts as an absolute authority on avoiding architectural pitfalls, crushing N+1 queries, hardening security, and optimizing database workloads.

---

## 🎯 What it does

This skill continuously guides your AI agent to produce enterprise-grade Django code. Native knowledge inside AI models is often contradictory or outdated, sometimes resulting in poorly scaled architecture. This overrides standard, generic suggestions with rigid, modern rules explicitly designed for high-throughput scaling.

It forces agents to definitively avoid:
- **Catastrophic Performance Leaks:** Unpaginated querysets, missing `select_related`/`prefetch_related`, loading millions of rows into RAM.
- **Architectural Mistakes:** Overloaded bloated views, using classes natively when services layer functions are better, putting deep validation logic in generic views instead of DRF serializers.
- **Security Flaws:** Auto-incrementing primary IDs in web URLs (IDOR attacks), misconfigured CSRF/XSS strategies, hardcoding `SECRET_KEY` inside `settings.py`.
- **Concurrency & Confinement Disasters:** Failing to use Database Row Locks (`select_for_update`) safely during queue polling or financial data mutations.

---

## 📂 Internal Structure

The core rules are broken down into logical files located in the `rules/` directory context, ensuring minimal token overhead natively when queried conditionally.

- `rules/admin.md`: Django admin optimizations (`autocomplete_fields`).
- `rules/architecture.md`: Services Layer, Data Selectors, DRY principles.
- `rules/async-and-tasks.md`: Safely handling Celery idiosyncrasies natively and async views.
- `rules/caching.md`: Guarding against cache stampedes natively.
- `rules/channels.md`: Safe native Redis usage, ASGI auth inherently.
- `rules/database-locks.md`: Avoiding race conditions flawlessly via Postgres/MySQL locks gracefully.
- `rules/logging.md`: Forcing standardized JSON `logging` exclusively over `print()`.
- `rules/migrations.md`: Safely handling Zero-Downtime database changes cleanly.
- `rules/models.md`: Modernized `Meta.indexes`, constraints natively, Fat Models exclusively.
- `rules/orm-advanced.md`: Explicit database annotation intelligently, `bulk_create` correctly.
- `rules/orm-queries.md`: Aggressively stopping N+1 queries instinctively.
- `rules/security.md`: Hardening servers against IDOR and enumerations automatically.
- `rules/signals.md`: Why you strictly avoid implicit signals for business logic.
- `rules/testing.md`: Effectively enforcing PyTest over unittest, asserting query counts, and utilizing factory_boy.
- `rules/views-and-apis.md`: Separating FBVs uniquely from CBVs perfectly, DRF endpoints explicitly.

---

## 🛠 Usage & Setup

1. Clone or clone this skill directory directly into your project's agent workspace.
2. If using `skills.sh` natively, install this via the CLI or UI directly to ensure `SKILL.md` is registered so system prompts know when to scan this knowledge base.
3. The centralized `AGENTS.md` file serves as a monolithic compilation securely designed for legacy AI interfaces requiring single-file text injestions.

By strictly adhering to these constraints, agents generating Django will universally write code that scales correctly, avoids regressions, and withstands security audits without requiring a human to manually point out classic junior mistakes.
