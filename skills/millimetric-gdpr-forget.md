---
name: Honor a GDPR/CCPA right-to-be-forgotten request
description: Permanently delete every event for a user in a Millimetric project using a secret key.
api: openapi/millimetric-openapi.yml
operations: [forget, queryEvents]
auth: Bearer sk_live_* ONLY (pk_* is explicitly rejected)
---

# Honor a GDPR/CCPA right-to-be-forgotten request

Use this skill to delete a user's data on request.

## Auth
- `forget` accepts **only** an `sk_live_*` (secret) key. A `pk_live_*` key returns `403 forget_requires_secret_key` — browser keys are rejected so a leaked key cannot wipe data.

## Steps
1. **(Optional) Confirm scope** — `queryEvents` (`GET /v1/query`) with `user_id=<id>` and a wide `from`/`to` to see what will be deleted.
2. **Delete** — `forget` (`POST /v1/forget`) with body `{ "user_id": "<id>" }`. Response `{ "ok": true, "queued": true }`. The mutation is queued on ClickHouse (`ALTER TABLE … DELETE`) and usually completes within seconds.
3. **Record the request** — the API does not yet write a structured audit row (roadmap), so log the call from your own audited context.

## Rules
- Deletion is scoped to the project owning the key and to rows where `user_id` matches.
- Anonymous events emitted before `identify` (no `user_id`) are NOT deleted — they are indistinguishable from other anonymous traffic. To forget by `anonymous_id`, run the equivalent ClickHouse `ALTER TABLE events DELETE` directly.
- Aggregated rollups (`daily_rollup`, `sessions`) are not rewritten; re-aggregate from surviving raw events if you must strip contributions.
- `500 forget_failed` means ClickHouse rejected the mutation — retry and check Worker logs.
