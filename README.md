# Skills

This repository contains reusable agent skill definitions. Each skill packages a focused set of instructions for a specific task or domain.

## Included Skills

### `modern-python-syntax`

Guidance for writing and reviewing Python code that targets Python 3.11 through 3.14.

It helps with steering agents towards using modern Python features and idioms, while avoiding outdated practices.

```shell
npx skills@latest add kubazarz/skills/modern-python-syntax
```

### `my-stack`

Opinionated default tech stack for greenfield projects: Python (FastAPI or
Django), Next.js + shadcn/ui, PostgreSQL, framework-native auth, uv + ruff +
pytest, and Railway/Hetzner deployment.

It steers new projects toward consistent, pre-decided choices while flagging the
few questions (auth strategy, Django User model) that must be asked per project.

```shell
npx skills@latest add kubazarz/skills/my-stack
```
