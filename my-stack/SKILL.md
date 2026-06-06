---
name: my-stack
description: The preferred default tech stack for starting new projects. Use when the user is starting a new project, scaffolding a codebase, asking "what stack should I use", setting up a new backend/frontend/app, or deciding on frameworks, database, auth, or deployment for a greenfield project. Covers Python (FastAPI/Django), Next.js + shadcn/ui frontend, PostgreSQL, framework-native auth, and Railway/Hetzner deployment. Skip when working inside an existing codebase that has already made these choices.
metadata:
  version: 1.0.0
  category: scaffolding
---

# My Stack

## Purpose

A set of opinionated, pre-decided defaults for greenfield projects so that new
codebases start from a consistent, well-understood foundation. When the user
starts something new, reach for these choices instead of re-litigating them —
but **ask the handful of questions flagged below**, because they genuinely
depend on the project.

## How to use this skill

1. Work out **backend shape** (FastAPI vs Django) from project size/complexity.
2. **Ask the per-project questions** at the start (auth strategy; Django User
   model identity). Don't assume — these are explicit decision points.
3. Apply the fixed defaults (language, package manager, DB, frontend, lint).
4. Choose deployment + local-dev shape based on the number of moving parts.

Do not silently substitute trendier alternatives (Vercel, Clerk, Auth.js,
Prisma, Drizzle, Poetry). These were considered and rejected on purpose — see
the rationale notes. If the user explicitly wants something else, follow them.

---

## Backend

**Language: Python.** Always.

### FastAPI vs Django — decide by size

| Project shape | Use |
|---|---|
| Small / focused service, API-first, few models | **FastAPI** |
| Large / feature-rich, admin needs, many models, "a product" | **Django** |

If it's ambiguous, ask: *"Is this a small focused service or a larger product
with an admin and lots of models?"*

### FastAPI branch

- **ORM: SQLModel** — merges SQLAlchemy + Pydantic v2, so models double as
  schemas.
- **Migrations: Alembic.**
- **Validation: Pydantic v2** (comes with SQLModel).

### Django branch

- **REST layer: Django Ninja** — Pydantic v2 schemas, FastAPI-like ergonomics.
  Prefer this over Django REST Framework.
- **Custom User model: ALWAYS.** Never ship Django's default `User`. Create an
  `AbstractUser` subclass in a dedicated app before the first migration (it is
  painful to change later).
  - **ASK at project start:** *"Username-based or email-only login?"* This
    decides whether you keep `username` or set `USERNAME_FIELD = "email"` and
    drop `username`.

### Shared backend defaults

| Concern | Choice |
|---|---|
| Package / env manager | **uv** (not pip/Poetry/pipenv) |
| Linter + formatter | **ruff** |
| Testing | **pytest** |
| Database | **PostgreSQL** |

Use the `uv` skill for project setup commands and the `ruff` skill for
lint/format details when scaffolding.

---

## Auth

- **Always use framework-native auth.** Django's auth system; FastAPI's own
  dependency-based auth (e.g. OAuth2 password/bearer flows). **No Clerk, no
  Auth.js/NextAuth, no third-party auth SaaS.**
- **ASK at project start:** *"What auth strategy — cookie session, JWT, OAuth
  providers, or a mix?"* This is project-dependent; do not default silently.

---

## Frontend

| Concern | Choice |
|---|---|
| Framework | **Next.js** (default) |
| Language | **TypeScript** |
| Components | **shadcn/ui** |

- Use the `shadcn` skill when adding/initialising components.
- **Customise the colour scheme** (and ideally radius/typography) so the app
  doesn't have the generic default-shadcn look. Set this up early via
  `components.json` / CSS variables rather than retrofitting.

---

## Local development

- **Run the app natively** with `uv run` against a **local PostgreSQL** by
  default. Keep it simple.
- **Reach for Docker Compose only when the project has multiple services** —
  e.g. a queue (Redis/RabbitMQ), cache, background workers, search. A single
  app + DB does **not** need Compose.

---

## Deployment

| Situation | Target |
|---|---|
| Small project / prototype | **Railway** — predictable pricing, not serverless |
| Production-serious | **Hetzner VPS + Docker Compose** |

- **Avoid Vercel and other serverless providers** — cost is unpredictable at
  scale. This is a deliberate constraint, even for the Next.js frontend.

### CI

- **Skip CI by default.** Add it when the project actually needs it (team grows,
  releases matter, flaky deploys). Don't scaffold pipelines into a prototype.

---

## Recommended `ruff` config

Drop this into `pyproject.toml`. Default is `py314` — tune `target-version`
down if the project targets an earlier version (check `requires-python`). This is a clean, opinionated baseline —
verify against current ruff docs (use the `ruff` skill) when scaffolding, since
rule codes evolve.

```toml
[tool.ruff]
target-version = "py314"
line-length = 120
src = ["src", "app"]

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "C4",   # flake8-comprehensions
    "SIM",  # flake8-simplify
    "TID",  # flake8-tidy-imports
    "RUF",  # ruff-specific rules
]
ignore = [
    "E501",  # line length handled by the formatter
]

[tool.ruff.lint.isort]
known-first-party = ["app"]

[tool.ruff.format]
quote-style = "double"
docstring-code-format = true
```

For Django projects, also consider Django-specific ignores as they surface
(e.g. `RUF012` on mutable class attributes for model `Meta`/serializers).

---

## Quick checklist for a new project

1. Backend small or large? → FastAPI or Django.
2. **Ask:** auth strategy?
3. Django? → **Ask:** username or email-only User; create custom `AbstractUser`.
4. `uv init`, add ruff + pytest, drop in the ruff config above.
5. PostgreSQL — local for dev, native `uv run` unless multi-service (then Compose).
6. Frontend needed? → Next.js + TypeScript + shadcn/ui with a custom theme.
7. Deploy: Railway (small) or Hetzner + Compose (serious). Not serverless.
8. CI only if the project needs it.
</content>
</invoke>
