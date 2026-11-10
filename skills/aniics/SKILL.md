---
name: aniics
description: Guidance for integrating, operating, and extending the ANIICS Cloudflare Workers analytics service. Use when an agent needs to instrument an app against ANIICS, emit analytics events to `/api/v1/p`, secure or query the `/mothersh1p` dashboard, adjust the Analytics Engine schema, or work on the ANIICS service implementation.
license: MIT
metadata:
  author: eltonstewart
  version: '1.2.0'
---

# ANIICS

Use this skill when the task is specifically about the ANIICS analytics service or integrating another project with it.

## Orient Fast

- Start from the service README or equivalent docs for the route model and security assumptions.
- Treat ANIICS as hybrid in production: public browser ingestion is allowed only for explicit low-value event/app allowlists, while sensitive events are sent by a signed backend.
- Identify the event schema and Analytics Engine field mapping before changing instrumentation. Analytics Engine allows **one index and at most 20 blobs per data point** — the event model is budgeted against that.
- Identify the ingestion, ingestion-auth, dashboard auth, analytics query, and dashboard modules before making service changes.
- Confirm whether the task is app integration work, ingestion/schema work, or dashboard/query work.

## Use Cases

- Add ANIICS tracking to another app or service
- Change ingestion validation, app allowlists, or origin checks
- Extend the event schema or analytics queries
- Work on the internal dashboard or its auth model
- Debug tests around ingestion, auth, schema, or dashboard behavior

## Working Rules

1. Preserve the current service shape unless the user explicitly wants an architecture change.
2. Treat Analytics Engine's single-index and 20-blob limits as hard design inputs. Only this layout is stored. `blob19` is `local|device|language|browser|surface`. The dashboard groups the whole blob and splits segments in the worker. Nested `substring` SQL can fail the SQL API. Do not store raw viewport or userAgent.

   | Slot   | Field       | Slot   | Field                                       |
   | ------ | ----------- | ------ | ------------------------------------------- |
   | index  | `appId`     | blob11 | `section`                                   |
   | blob1  | `event`     | blob12 | `ctaId`                                     |
   | blob2  | `timezone`  | blob13 | `term`                                      |
   | blob3  | `pathname`  | blob14 | `content`                                   |
   | blob4  | `sessionId` | blob15 | `interaction`                               |
   | blob5  | `userId`    | blob16 | `target`                                    |
   | blob6  | `variant`   | blob17 | `targetId`                                  |
   | blob7  | `source`    | blob18 | `action`                                    |
   | blob8  | `medium`    | blob19 | `local\|device\|language\|browser\|surface` |
   | blob9  | `campaign`  | blob20 | `href`                                      |
   | blob10 | `referrer`  |        |                                             |

   `device` is `mobile` under 768px, `tablet` under 1024px, otherwise `desktop`. `browser` is `chrome`, `safari`, `firefox`, `edge`, `opera`, `samsung`, or `other`. Empty segments stay empty. A `|` in a value is stored as `/`.

3. Keep ingestion hardened: strict validation, bounded fields, fail-closed behavior, signed sender verification, event allowlists, and explicit public exceptions only.
4. Prefer changes that keep client integration simple and server secrets server-side.
5. Do not put ANIICS signing secrets in browser code or public bundles.

## Production Auth Model

- Production ingestion is hybrid: GA-style browser ingestion for allowed low-value events, signed HMAC ingestion for sensitive/server events.
- Apps are registered one of two ways, with one gate each:
  - **Workspace registry (tenant apps)**: created in the dashboard (`/onboarding` or app management). `public` mode requires allowed origins, enables a per-app public event list, and returns a one-tag snippet — no secret. `server` mode returns an HMAC secret that is AES-GCM encrypted at rest (`APP_SECRET_ENCRYPTION_KEY`) and rotatable; old secrets are deactivated.
  - **Platform apps (env allowlist)**: for the service's own sites; gated by `ANALYTICS_ALLOWED_APP_IDS` and the worker-wide origin list.
- Tenant apps are gated by their own allowed origins and workspace entitlement; they are never required to also appear in the worker-wide lists. Apps not in the registry fall back to the env allowlist.
- Dashboard HTML (`/mothersh1p`) is a public login shell; data (`/mothersh1p/api/stats`) requires a Better Auth session and is scoped to the session's workspace.
- Trusted senders must include `X-Aniics-App`, `X-Aniics-Timestamp`, and `X-Aniics-Signature`.
- The signature is computed over `timestamp + "." + raw_body`.
- Verify signatures against the raw request body before JSON parsing.
- Enforce timestamp freshness with `ANALYTICS_SIGNED_REQUEST_MAX_AGE_SECONDS`, currently expected to be short such as 300 seconds.
- Use `ANALYTICS_REQUIRE_SIGNED_INGESTION=true` in production, with `ANALYTICS_UNSIGNED_PUBLIC_APP_IDS` listing only apps allowed to send low-trust browser telemetry.
- Keep `ANALYTICS_PUBLIC_EVENT_NAMES` restricted to low-value events.
- Keep `ANALYTICS_SIGNED_EVENT_NAMES` covering conversions, identity, purchases, checkout, revenue, and other sensitive events.
- Use `ANALYTICS_MAX_BODY_BYTES` to cap request size before expensive processing.
- Direct browser-to-ANIICS posting is never cryptographic auth; rely on origin checks, event allowlists, app allowlists, body caps, WAF/rate limits, and signed-only sensitive events to minimize abuse.
- Prefer structured behavior events (`view`, `interaction`, `state_change`, `submission`, `error`) with dimensions instead of many bespoke event names.

## Production Secrets

- `ANALYTICS_INGESTION_SECRETS`: JSON object mapping `appId` to HMAC secret.
- `ANALYTICS_REQUIRE_SIGNED_INGESTION`: set to `true` for production enforcement.
- `ANALYTICS_UNSIGNED_PUBLIC_APP_IDS`: comma-separated app ids allowed to send low-trust public browser events.
- `ANALYTICS_PUBLIC_EVENT_NAMES`: comma-separated low-risk events allowed from unsigned browsers. The platform default is the generic visitor-behaviour list only (`page_view`, `cta_click`, `view`, `interaction`, `state_change`, `submission`, `error`, `pointer_engaged`, `page_engaged`); app-specific events belong in the app's own list.
- `ANALYTICS_SIGNED_EVENT_NAMES`: comma-separated events that must be signed even for public apps.
- `ANALYTICS_REQUIRE_ORIGIN_FOR_UNSIGNED`: set to `true` if unsigned exceptions exist.
- `ANALYTICS_MAX_BODY_BYTES`: request body cap; `8192` is the current documented default.
- `ANALYTICS_SIGNED_REQUEST_MAX_AGE_SECONDS`: replay window; `300` is the current documented default.
- `DASHBOARD_SHARED_SECRET`: legacy fallback dashboard secret for non-DB deployments; session auth (Better Auth) is the multi-user path.
- `BETTER_AUTH_SECRET` / `BETTER_AUTH_URL`: session auth; set `BETTER_AUTH_URL` to your https origin so session cookies are `Secure`.
- `APP_SECRET_ENCRYPTION_KEY`: encrypts per-app HMAC secrets at rest in D1; set before issuing any server-mode app secrets.
- `PLATFORM_ADMIN_USER_IDS`: comma-separated user ids with platform admin access.

## Task Routing

### Integrating Another App

- Start with the public integration docs or client helper if one exists.
- For a public marketing site, prefer the one-tag collector: `<script src="https://aniics.fyi/aniics.js" data-app-id="…" defer></script>` — auto page views (including SPA history navigations), one engagement event per page, and delegated clicks (downloads, outbound links, `data-cta` elements). It sends `text/plain` beacons (no CORS preflight) and holds no secrets. Mark CTAs with `data-cta="id"` plus optional `data-cta-action`, sections with `data-section`.
- For low-value browser analytics beyond the tag, confirm the app is in `ANALYTICS_UNSIGNED_PUBLIC_APP_IDS` and its events are in `ANALYTICS_PUBLIC_EVENT_NAMES`.
- Model behavior with stored columns: `event`, `pathname`, `section`, `ctaId`, `interaction`, `target`, `targetId`, `action`, and `href`. Put the flow in `context` and the UI region in `surface`. Those two are segments of blob19, not their own columns.
- `environment`, `label`, `affected`, `screen`, and `properties` are accepted and not stored. Raw viewport and userAgent are accepted only to derive `device` and `browser`. Use a separate `appId` per environment. Do not build charts on `properties`.
- Capture traffic origin with allowlisted query params (`utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content`, `ref`, and common click ids) plus the landing referrer. Keep those values for the session; replace them when a later URL has new marketing params. Never store the raw query string. Click ids are not stored.
- Model origin with `source`, `medium`, `campaign`, `term`, `content`, and `referrer`. Language, device, and browser are segments of blob19.
- For sensitive analytics, route browser events to the app's own backend endpoint, then forward to ANIICS with signed headers.
- Use `client/aniics-server.ts` as the server-side signing helper when appropriate.
- Confirm the app sends `appId` and `event`, with optional fields that already map cleanly into the current event schema.
- Confirm the app's `appId` is present in both `ANALYTICS_ALLOWED_APP_IDS` and `ANALYTICS_INGESTION_SECRETS` before requiring signed ingestion (platform apps), or registered as a workspace app (tenant apps).
- Ingestion rejections return distinguishable codes: `unknown_app`, `origin_not_allowed`, `event_not_public`, `signed_required`, `origin_required`, `subscription_required`.
- If a new client-side field is required, update schema, ingestion, and tests together.

### Schema or Ingestion Changes

- Update schema and ingestion logic together.
- Keep field naming and limits consistent with the documented event model.
- Preserve raw-body HMAC verification before parsed JSON validation.
- Keep signed app header and JSON body `appId` aligned; mismatches must be rejected.
- Run the schema, ingestion-auth, and ingestion tests after changes.

### Dashboard or Query Changes

- Update dashboard and analytics query logic together. Queries read SQL columns `blob1`–`blob20` per the documented mapping — when the schema changes, remap every query and the dashboard cards.
- Preserve the existing auth boundary: public login shell, session-gated and workspace-scoped stats API.
- Prefer explicit, auditable server-side query behavior over convenience shortcuts.

## Validation

- Run `bun run test` for broad verification.
- Run `bun run check` when changing TypeScript, lint-sensitive code, or docs referenced by the skill.
