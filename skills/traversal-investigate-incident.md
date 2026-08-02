---
name: traversal-investigate-incident
description: >-
  Launch a Traversal AI-SRE investigation for a production incident, poll it to
  completion, and read the root-cause result — then optionally drill in with a
  follow-up. Grounded in the real Traversal V1 Sessions API operations.
generated: '2026-07-21'
method: generated
api: Traversal Sessions API
source: openapi/traversal-sessions-openapi.yaml
operations:
  - createSession
  - getSession
  - sendFollowUpMessage
  - listSessions
---

# Investigate a production incident with Traversal

Traversal runs investigations asynchronously: you start one, it works in the
background, and you poll until it is `idle` to read the result.

## Prerequisites

- An API key from **Settings > API keys** in the Traversal web app
  (`app.traversal.com/settings/api-keys`). Pass it as
  `Authorization: Bearer trv_ak_...`.
- The V1 API must be enabled for your organization (otherwise every call
  returns `403 Forbidden`).
- Base URL `https://api.traversal.com` (or your dedicated single-tenant / BYOC
  endpoint).

## Steps

1. **Start the investigation** — `createSession`
   (`POST /v1/sessions`). Send the incident description in `input`, an
   optional `time` (ISO-8601 of when it started), an optional `title`, and a
   **required** `idempotency_key`. Tie the key to the upstream event (e.g. a
   PagerDuty incident ID) so retries never create duplicate sessions —
   resubmitting the same key returns the original session with `200` instead of
   `201`. Optionally set `thinking_mode` to `deep`, `fast`, or leave it `auto`.
   Capture the returned session `id`.

2. **Poll until complete** — `getSession`
   (`GET /v1/sessions/{session_id}`). Repeat until `status` is `idle`. This is
   the only endpoint that populates the `messages` array; read the latest
   `assistant` message's `markdown_content` for the root-cause summary,
   evidence, and recommendations. If `status` becomes `failed`, the
   investigation errored or timed out.

3. **Drill deeper (optional)** — `sendFollowUpMessage`
   (`POST /v1/sessions/{session_id}/messages`). The session must be `idle` or
   you get `409 Conflict` with `retry_after`. Send your follow-up in `input`;
   the response is `202` with `assistant_message_id`. Poll `getSession` again
   until `idle` to read the answer. Follow-ups use the full conversation as
   context.

4. **Review across the org (admin)** — `listSessions`
   (`GET /v1/sessions`). Admin-role keys only; lists V1-API sessions across the
   organization with page-number pagination (`page`, `limit`; `prev` / `next` /
   `total`).

## Rules and gotchas

- **Concurrency:** an org may have at most **15 concurrent running sessions**
  (new investigations and in-flight follow-ups both count). Over the limit you
  get `429 Too Many Requests` with `retry_after` — honor the `Retry-After`
  header before retrying.
- **Error envelope:** `{ "error": { "message", "retry_after" } }` — not RFC 9457.
- **Roles:** most endpoints need `member`; `listSessions` needs `admin`.
- Read-only by design: Traversal never modifies your systems.
