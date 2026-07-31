---
name: Stitch anonymous visitors to known users
description: Link an anonymous_id to a user_id with Millimetric, then query that user's events across the pre/post-login boundary.
api: openapi/millimetric-openapi.yml
operations: [trackEvent, identify, queryEvents]
auth: Bearer sk_live_* to ingest/identify, Bearer rk_live_* to query
---

# Stitch anonymous visitors to known users

Use this skill to connect pre-login (anonymous) activity to a known user, then read it back.

## Auth
- `trackEvent` and `identify` need an **ingest** key (`sk_live_*` server-side, or `pk_live_*` in the browser with an allowlisted `Origin`).
- `queryEvents` needs a **read** key (`rk_live_*`).

## Steps
1. **Track anonymously** — `trackEvent` (`POST /v1/track`) with a stable `anonymous_id` while the visitor is logged out (`event: "$pageview"`, CTA clicks, etc.).
2. **Identify on login/signup** — `identify` (`POST /v1/identify`) with the required `anonymous_id` and `user_id`, plus optional `traits`. This persists a `$identify` event as the stitch point; historical anonymous events are NOT rewritten.
3. **Tag subsequent events** — keep calling `trackEvent` but now include `user_id` on each call (the browser/Node SDK does this automatically after `identify()`).
4. **Read a user's history** — `queryEvents` (`GET /v1/query`) with required `from`/`to` and `user_id=<id>`, `limit` up to 1000. `properties` comes back as a JSON string — parse client-side.

## Rules
- `identify` requires BOTH `anonymous_id` and `user_id` (1–128 / 1–256 chars).
- Anonymous events emitted before `identify` have no `user_id` and cannot be retroactively attributed by `user_id`.
- Do not send PII in event `properties`; `traits` on `$identify` is the place for user attributes.
- Errors follow errors/millimetric-error-codes.yml; auth/scope mistakes return 401/403 with a stable code.
