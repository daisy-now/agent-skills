# Daisy API reference

The lookup appendix for the Daisy REST API. Read [../SKILL.md](../SKILL.md) first for the workflow. Everything here is under `https://www.daisy.now/api`. Replace `:id` with project id, `:sid` with screen id, `:runId` with run id.

## Contents

- Endpoint table
- Response envelope
- Scopes & plans
- Idempotency
- Errors
- Rate limits
- Credits
- Hard limits

## Endpoint table

| Method | Path | Scope | Credits |
|---|---|---|---|
| GET | `/user/me` | any key | 0 |
| GET | `/user/usage?days=&projectId=` | any key | 0 |
| GET, POST | `/user/api-keys` | session/key | 0 |
| DELETE | `/user/api-keys/:id` | session/key | 0 |
| GET | `/projects` | `projects:read` | 0 |
| POST | `/projects` _(idempotent)_ | `projects:write` | 0 |
| GET | `/projects/:id` | `projects:read` | 0 |
| PATCH | `/projects/:id` _(idempotent)_ | `projects:write` | 0 |
| DELETE | `/projects/:id` | `projects:write` | 0 |
| GET | `/projects/:id/screens` | `screens:read` | 0 |
| GET | `/projects/:id/screens/:sid` | `screens:read` | 0 |
| POST | `/projects/:id/screens` _(idempotent)_ | `screens:write` | **yes** |
| PATCH | `/projects/:id/screens` | `screens:write` | 0 |
| DELETE | `/projects/:id/screens` | `screens:write` | 0 |
| POST | `/projects/:id/runs` _(idempotent)_ | `runs:write` | **yes** |
| GET | `/projects/:id/runs` | `runs:read` | 0 |
| GET | `/projects/:id/runs/:runId` | `runs:read` | 0 |
| DELETE | `/projects/:id/runs/:runId` | `runs:write` | 0 |
| POST | `/screenshots` | `screenshots` | 0 |

Single-screen `GET /screens/:sid` carries the rendered `html`. The list `GET /screens` returns metadata only (`id`, `label`, `status`).

## Response envelope

- Collections → `{ "data": [ … ], "cursor": null }`
- Single resources → the object directly
- Simple mutations → `{ "ok": true, … }`
- Screenshots → raw image bytes (`Content-Type: image/png` or `image/jpeg`)

## Scopes & plans

Scopes use colon form. A key carries a subset; a request missing the scope returns `403 forbidden:api`.

| Scope | Grants |
|---|---|
| `projects:read` | List and read projects |
| `projects:write` | Create, update, delete projects |
| `screens:read` | Read screen metadata and HTML |
| `screens:write` | Create, batch-edit, and delete screens directly |
| `runs:read` | Read run status and output |
| `runs:write` | Send messages / start and cancel runs |
| `screenshots` | Render a screen to an image |

Meta-scopes expand at mint time: `apis:all` → every scope; `apis:read` → every `*:read` scope. Presets: **`full_access`** (all) and **`read_only`** (`projects:read`, `screens:read`, `runs:read`).

For an agent that inspects but never spends credits, mint a **read_only** key. Free/Plus plans cannot call the API at all (`403`); generation needs **Pro or Max**.

> Legacy dotted scopes (`projects.read`, `chat.write`, …) are mapped on read for old keys, but mint new keys with the colon names. `themes.*` and `usage.*` no longer map to anything.

## Idempotency

Endpoints marked _(idempotent)_ accept an `Idempotency-Key` header (use a UUID). A retry with the **same key and body** replays the original response with `X-Idempotent-Replay: true` instead of running twice. A different body under the same key is a new slot.

**Reuse the key across retries** — that is the entire point. Retry (same key) on:

- `429 rate_limit:api` — wait the `Retry-After` seconds.
- `500 internal_error` — exponential backoff (1s, 2s, 4s, … cap ~30s).
- Network / transport errors.

Never retry other `4xx` — the request itself is wrong.

## Errors

Every error is JSON: `{ "code": "...", "message": "..." }` with a matching status.

| Code | Status | When | What to do |
|---|---|---|---|
| `unauthorized:api` | 401 | Missing, malformed, revoked, or unknown key | Re-create the key in the dashboard |
| `forbidden:api` | 403 | Missing scope, wrong plan, or exhausted credits | Upgrade, fix billing, or mint a key with the scope |
| `bad_request:api` | 400 | Invalid body or query | Fix the request — don't retry |
| `not_found:api` | 404 | Resource doesn't exist or isn't yours | Don't retry — the id is wrong |
| `conflict:api` | 409 | Active run on the project, or screenshot of an unrendered screen | Cancel/await the run; wait for screen status `done` |
| `rate_limit:api` | 429 | Rate-limited | Honor `Retry-After` |
| `internal_error` | 500 | Unexpected server error | Retry with backoff |

## Rate limits

Rate limits apply to **API keys only** (session traffic is exempt). Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. On `429`, `Retry-After` gives the exact seconds — honor it.

## Credits

Only `POST /screens` and edits made inside a run consume credits. Reads, deletes, batch field updates, theme changes, and screenshots are free. The API enforces your plan's allowance and returns `403`/`409` when credits run out.

Don't guess cost — each run reports the real `creditsCharged` in its terminal object. Check the balance any time:

```bash
curl -s "https://www.daisy.now/api/user/me" \
  -H "Authorization: Bearer $DAISY_API_KEY" | jq '{plan, credits}'
```

## Hard limits

- One active run per project (`409` on a concurrent `POST /runs`).
- `label`: 1–120 chars · `brief`: 20–4000 chars · `message`: 1–4000 chars.
- Batch screen update/delete: 1–500 items per request.
- Only screens with status `done` can be screenshotted (`409` otherwise).
