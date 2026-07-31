---
name: Instrument a product and report on traffic sources
description: Emit analytics events to Millimetric and then report aggregate stats and the paid-vs-social attribution split.
api: openapi/millimetric-openapi.yml
operations: [trackEvent, batchEvents, getStats, getSources]
auth: Bearer sk_live_* to ingest, Bearer rk_live_* to read (never a pk_* key server-side)
---

# Instrument a product and report on traffic sources

Use this skill to send events into a Millimetric project and then answer traffic/attribution questions.

## Auth
- Ingest (`trackEvent`, `batchEvents`) needs an **ingest** scope key: `sk_live_*` on servers.
- Reads (`getStats`, `getSources`) need a **read** scope key: `rk_live_*`.
- Never use a `pk_live_*` key server-side — it is origin-allowlisted and browser-only.
- Header: `Authorization: Bearer {key}`.

## Steps
1. **Emit a single event** — `trackEvent` (`POST /v1/track`). Required body field: `event`. Optional: `anonymous_id`, `user_id`, `url`, `referrer`, `properties`, and `event_id` (idempotency/join key — note there is no server-side dedup yet). The server adds source/medium/geo/device itself; do not send them.
2. **Emit many at once** — `batchEvents` (`POST /v1/batch`) with `{ "events": [...] }`, up to 1000. All-or-nothing: if one event fails validation, none are written. Prefer this for backfills.
3. **Aggregate** — `getStats` (`GET /v1/stats`) with `metric=count|uniques`, required `from`/`to` (ISO 8601, `to` exclusive), optional `group_by` (only `event_name,source,medium,country,device_type,browser,os,path`) and `interval=hour|day|week`.
4. **Attribution split** — `getSources` (`GET /v1/sources`) with `from`/`to` and `breakdown=source_medium|source`. Each row has `events`, `uniques`, and `paid_share` — this surfaces the Facebook social-vs-paid split.

## Rules
- Rate limits: `/v1/track` 50/sec (burst 200), `/v1/batch` 5/sec (burst 20). On `429 rate_limited`, honour the `Retry-After` header; do not tight-retry.
- Errors are `{ "error": "<code>" }` with a stable code; `400` = fix the payload (no retry), `5xx` = exponential backoff. See errors/millimetric-error-codes.yml.
- `group_by` with an unknown column returns `400 invalid_group_by` with the allowed list.
