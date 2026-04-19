---
name: aniics
description: Guidance for integrating, operating, and extending the ANIICS Cloudflare Workers analytics service. Use when an agent needs to instrument an app against ANIICS, emit analytics events to `/api/v1/p`, secure or query the `/mothersh1p` dashboard, adjust the Analytics Engine schema, or work on the ANIICS service implementation.
license: MIT
metadata:
  author: aniics
  version: "1.0.0"
---

# ANIICS

Use this skill when the task is specifically about the ANIICS analytics service or integrating another project with it.

## Orient Fast

- Start from the service README or equivalent docs for the route model and security assumptions.
- Identify the event schema and Analytics Engine field mapping before changing instrumentation.
- Identify the ingestion, auth, analytics query, and dashboard modules before making service changes.
- Confirm whether the task is app integration work, ingestion/schema work, or dashboard/query work.

## Use Cases

- Add ANIICS tracking to another app or service
- Change ingestion validation, app allowlists, or origin checks
- Extend the event schema or analytics queries
- Work on the internal dashboard or its auth model
- Debug tests around ingestion, auth, schema, or dashboard behavior

## Working Rules

1. Preserve the current service shape unless the user explicitly wants an architecture change.
2. Treat Analytics Engine's single-index constraint as a hard design input.
3. Keep ingestion hardened: strict validation, bounded fields, fail-closed behavior.
4. Prefer changes that keep client integration simple and server secrets server-side.

## Task Routing

### Integrating Another App

- Start with the public integration docs or client helper if one exists.
- Confirm the app sends `appId` and `event`, with optional fields that already map cleanly into the current event schema.
- If a new client-side field is required, update schema, ingestion, and tests together.

### Schema or Ingestion Changes

- Update schema and ingestion logic together.
- Keep field naming and limits consistent with the documented event model.
- Run the schema and ingestion tests after changes.

### Dashboard or Query Changes

- Update dashboard and analytics query logic together.
- Preserve the existing auth boundary.
- Prefer explicit, auditable server-side query behavior over convenience shortcuts.

## Validation

- Run `bun run test` for broad verification.
- Run `bun run check` when changing TypeScript, lint-sensitive code, or docs referenced by the skill.
