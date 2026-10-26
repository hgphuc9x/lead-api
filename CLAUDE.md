# Project: leads-api

## Overview

A backend-only Sales Lead Management tool for car dealerships. Leads arrive from the dealership website (`POST /leads`, API key, idempotent), salespeople log in to view the lead inbox, see lead details with the chronological activity log, and record follow-up activities. Swagger UI is the only client. The design targets a modular monolith that is multi-tenant ready (every row belongs to a dealership).

## Tech Stack

- Backend: NestJS, TypeScript (strict), modular monolith
- Database: PostgreSQL with TypeORM (migrations only, `synchronize: false`)
- Cache / shared state: Redis (token blacklist, rate-limit counters, lead cache)
- Auth: JWT access (15 min) + refresh (7 days) tokens; API key for the website
- API docs: Swagger UI (`/docs`)
- Observability: nestjs-pino (JSON logs), Prometheus metrics, health probes
- Tests: Jest + Supertest

## Docs

Read them before answering or changing code:

- @docs/API_SPEC.md
- @docs/DATABASE.md
- @docs/BE-PROJECT-RULES.md
- @docs/BE-ARCHITECTURE.md

## Available Skills

- `/plan`: plan a feature before coding
- `/refactor`: refactor without changing behavior
- `/explain`: explain code, flow or decisions
- `/summarize-changes`: summarize uncommitted changes, flag risks
- `/git-commit`: write a conventional commit message
- `/git-resolve-conflict`: resolve merge conflicts

## Available Subagents

- `code-reviewer`: review changes for BE-PROJECT-RULES.md conventions, correctness, tests, API_SPEC.md / DATABASE.md updated (read-only)
- `code-security`: security audit for `dealership_id` scoping, auth/token handling, input validation, no PII in logs (read-only)

MUST: after every feature or bug fix, before reporting done, launch both subagents in parallel (one message) to check the change.

### Important

- Follow existing patterns in codebase
- Read BE-PROJECT-RULES.md before adding a feature or module
- Update DATABASE.md and API_SPEC.md in the same change as any entity or endpoint change
