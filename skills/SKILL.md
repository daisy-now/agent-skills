---
name: design-mobile-apps-with-daisy
description: Design native mobile app screens via the Daisy REST API. Use this when a user asks you to mock up, design, prototype, or iterate on mobile app screens — onboarding flows, settings pages, home feeds, dashboards, checkout, sign-up, etc. Drives the daisy.now canvas over HTTPS, returns each screen's rendered HTML, and can render screens to PNG/JPEG images.
---

# Design mobile app screens with Daisy

Daisy (https://www.daisy.now) is an AI canvas for designing native mobile app screens. This skill drives Daisy's REST API at `https://www.daisy.now/api` so you can spin up projects, generate screens, iterate on them, read the rendered HTML, and render screens to images — all from inside your coding agent.

The API is **unified and versionless**: there is no `/v1` prefix, and every endpoint lives under `https://www.daisy.now/api`.

## When to use

Use this skill when the user says things like:

- "Design me a settings screen for X"
- "Mock up an onboarding flow for my SaaS"
- "Build a home feed for a fitness app"
- "Iterate on this screen — make the CTA bigger"
- "Sketch the empty state for the inbox"
- "Add a checkout screen to the project we made yesterday"

Do NOT use this skill for: web pages (Daisy is mobile-only), design system tokens divorced from screens, or copywriting that doesn't ship pixels.

## Setup

The user must have a **Pro or Max** Daisy subscription and have created an API key in **Settings → API keys** at https://www.daisy.now/dashboard.

Read the key from the environment:

```bash
export DAISY_API_KEY="dsy_live_..."
```

The domain is always `https://www.daisy.now` — there is no base-URL override.

Every request authenticates with the key, sent as either header:

```
Authorization: Bearer $DAISY_API_KEY
# or
x-api-key: $DAISY_API_KEY
```

JSON requests also send `Content-Type: application/json`.

If `DAISY_API_KEY` is missing or starts with `dsy_test_`, fail early and ask the user to paste a live key. Never invent a key.

**Scopes.** A key only grants the scopes it was minted with; a request missing the required scope returns `403 forbidden:api`. The endpoint reference below lists the scope each route needs. If the user wants the agent to inspect but never spend credits, they should mint a **read_only** key (`projects:read`, `screens:read`, `runs:read`).

## Workflow A — Runs (recommended)

Use this when the user describes the work in natural language and you want Daisy to plan + generate everything in one call. **This is the default — prefer it over Workflow B.**

### 1. Make sure you have a project

```bash
# Reuse an existing project if the user already mentioned one
curl -s "https://www.daisy.now/api/projects" \
  -H "Authorization: Bearer $DAISY_API_KEY" | jq '.data[] | {id, name}'

# Or create a new one
curl -s -X POST "https://www.daisy.now/api/projects" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"idea":"fitness tracker for runners","name":"Runr"}'
# → 201 with the full project object: { "id": "abc123", "name": "Runr", ... }
```

### 2. Fire a run

```bash
curl -s -X POST "https://www.daisy.now/api/projects/abc123/runs" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "message": "Design onboarding (3 screens: welcome, goal-picker, permissions) and a home feed with today'\''s run summary.",
    "wait": "none"
  }'
# → 202 {
#     "runId": "run_xyz",
#     "status": "queued",
#     "projectId": "abc123",
#     "pollUrl": "/api/projects/abc123/runs/run_xyz"
#   }
```

`wait` values:

- `"none"` (default) — returns `202` immediately with `runId` + `pollUrl`. **Use this**; poll every 3-5s.
- `"block"` — keeps the request open until the run finishes, then returns the completed Run object. Only use when the user is watching your terminal and the run is small (1-2 screens).

Other body fields (optional):

- `modelKey` — admin model override.
- `tools` — `{ "mcp": false, "skills": false }` to disable the user's MCP servers / agent skills for this run. Both default on.

**Concurrency**: only one active run per project. A second concurrent POST returns `409 conflict:api`. Either wait for the first run, or cancel it (see step 6).

### 3. Poll until terminal

Poll the `pollUrl` returned by step 2 (it is the nested path `/api/projects/:id/runs/:runId`):

```bash
while true; do
  STATE=$(curl -s "https://www.daisy.now/api/projects/abc123/runs/run_xyz" \
    -H "Authorization: Bearer $DAISY_API_KEY")
  STATUS=$(echo "$STATE" | jq -r .status)
  case "$STATUS" in
    succeeded|failed|cancelled) echo "$STATE" | jq .; break ;;
  esac
  sleep 3
done
```

A Run object looks like:

```jsonc
{
  "id": "run_xyz",
  "projectId": "abc123",
  "status": "queued | running | succeeded | failed | cancelled",
  "input": { "message": "…", "modelKey": null, "enableMcp": true, "enableSkills": true },
  "output": {
    "summary": "2 screens created",
    "lastAssistantMessage": "Done. Onboarding starts on the welcome screen…"
  },
  "error": null,
  "operations": [
    { "type": "screen_created", "screenId": "scr_1", "label": "Welcome", "ok": true, "at": 0 },
    { "type": "screen_created", "screenId": "scr_2", "label": "Goal picker", "ok": true, "at": 0 }
  ],
  "createdAt": 0, "startedAt": 0, "finishedAt": 0, "abortRequestedAt": null,
  "creditsCharged": 20
}
```

### 4. Resolve screen IDs into rendered HTML

The run lists the screens it touched in `operations[].screenId`. Fetch each screen — the single-screen GET includes the rendered `html`:

```bash
for SID in $(echo "$STATE" | jq -r '.operations[].screenId'); do
  curl -s "https://www.daisy.now/api/projects/abc123/screens/$SID" \
    -H "Authorization: Bearer $DAISY_API_KEY" \
    | jq '{id, label, status, html}'
done
```

Or list all screens in the project (metadata only — no HTML):

```bash
curl -s "https://www.daisy.now/api/projects/abc123/screens" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  | jq '.data[] | {id, label, status}'
```

Each screen's `status` is `loading | done | error`. Only screens with status `done` have final HTML and can be rendered to an image.

### 5. Show the screens to the user

There are no preview URLs anymore — screens carry their HTML directly. Two ways to hand the user something they can open:

**a) Save the HTML to a file** and point them at it:

```bash
curl -s "https://www.daisy.now/api/projects/abc123/screens/scr_1" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  | jq -r '.html' > welcome.html
# → open welcome.html in a browser
```

**b) Render an image** via the screenshot endpoint (PNG by default):

```bash
curl -s -X POST "https://www.daisy.now/api/screenshots" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"projectId":"abc123","screenId":"scr_1","format":"png"}' \
  -o welcome.png
# → raw image bytes (Content-Type: image/png)
# 404 if the screen doesn't exist; 409 if it hasn't been rendered yet (wait for status "done")
```

Then print the local paths for the user:

```
✓ Welcome screen      → ./welcome.png
✓ Goal picker         → ./goal-picker.png
✓ Permissions prompt  → ./permissions.png
```

### 6. Cancel a run (if needed)

```bash
curl -s -X DELETE "https://www.daisy.now/api/projects/abc123/runs/run_xyz" \
  -H "Authorization: Bearer $DAISY_API_KEY"
# Queued/running → returns 202 with status "aborting"; poll until "cancelled"
# Already terminal → returns the existing run unchanged
```

## Workflow B — Direct screen ops (advanced)

Skip the run orchestrator when you want precise control — e.g. generating a single screen, repositioning the canvas, or applying an HTML edit you've authored yourself. These calls are synchronous.

### Create one screen

```bash
curl -s -X POST "https://www.daisy.now/api/projects/abc123/screens" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "label": "Settings",
    "brief": "iOS-style settings list: account section with avatar, then notifications, privacy, appearance (light/dark/auto), and a destructive Delete account at the bottom."
  }'
# → 201 with the created screen (includes html). Consumes credits.
```

`label` is short (1-120 chars, what the user sees). `brief` is the generation instruction (20-4000 chars; longer + more specific = better output).

### Edit a screen

There is **no per-screen AI edit endpoint** anymore. To change a screen:

- **Via natural language** — fire a run (Workflow A) with a message like *"On the Settings screen, make the primary CTA orange and double its size."* The agent edits the existing screen in place. This is the recommended path for design changes.
- **Via direct HTML** — if you already have the new markup, push it through the batch update below.

### Batch update screens

Applies field-level updates to many screens at once (canvas drag/resize, inline HTML edits):

```bash
curl -s -X PATCH "https://www.daisy.now/api/projects/abc123/screens" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "screens": [
      { "id": "scr_1", "x": 120, "y": 0, "userResized": true },
      { "id": "scr_2", "html": "<!doctype html>…" }
    ]
  }'
# → { "ok": true }
```

Each item is `{ id, x?, y?, height?, userResized?, html?, prompt?, generationPrompt?, status? }` (1-500 items).

### Batch delete screens

```bash
curl -s -X DELETE "https://www.daisy.now/api/projects/abc123/screens" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"ids":["scr_1","scr_2"]}'
# → { "ok": true } (1-500 ids)
```

### Set the theme

The theme lives on the project — update it with `PATCH /projects/:id` (there is no separate theme route):

```bash
curl -s -X PATCH "https://www.daisy.now/api/projects/abc123" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"theme":{"colors":{"primary":"#0A84FF","background":"#0B0F17","text":"#F2F4F7"},"radius":"medium"}}'
# → the updated full project
```

Set the theme **before** generating screens — every screen inherits from it. The current theme is included in `GET /projects/:id`.

## Showing screens in YOUR UI

If you're building a UI that wraps this skill (a desktop app, a web frontend), you have two embed options now that signed preview URLs are gone:

**Render the screen's HTML inline** with an `srcdoc` iframe:

```html
<iframe srcdoc="{escaped screen.html}" width="390" height="844" frameborder="0"></iframe>
```

**Or render to an image** via `POST /api/screenshots` and display the returned bytes (`<img src="data:image/png;base64,…">` or a saved file).

## Endpoint reference

All paths are under `https://www.daisy.now/api`. Replace `:id` with project ID, `:sid` with screen ID, `:runId` with run ID.

| Method | Path | Scope | Credits |
|---|---|---|---|
| GET | `/user/me` | any key | 0 |
| GET | `/user/usage?days=&projectId=` | any key | 0 |
| GET `POST` | `/user/api-keys` | session/key | 0 |
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

**Response envelope.** Collections return `{ "data": [ … ], "cursor": null }`. Single resources return the object directly. Simple mutations return `{ "ok": true, … }`.

## Scopes & plans

Scopes use colon form. A key carries a subset; a request missing the scope returns `403`.

| Scope | Grants |
|---|---|
| `projects:read` | List and read projects |
| `projects:write` | Create, update, delete projects |
| `screens:read` | Read screen metadata and HTML |
| `screens:write` | Create, batch-edit, and delete screens directly |
| `runs:read` | Read run status and output |
| `runs:write` | Send chat messages / start and cancel runs |
| `screenshots` | Render a screen to an image |

Meta-scopes expand at creation time: `apis:all` → every scope, `apis:read` → every `*:read` scope. Presets: **`full_access`** (all) and **`read_only`** (`projects:read`, `screens:read`, `runs:read`).

> Legacy dotted scopes (`projects.read`, `themes.*`, `chat.*`, `usage.*`) are mapped on read for old keys, but mint new keys with the colon names above.

Free/Plus tiers can't call the API — they get `403 forbidden:api`. Tell the user to upgrade at https://www.daisy.now/pricing.

## Credits

Screen generation (`POST /screens`) and edits made inside a run consume credits; reads, deletes, batch field updates, theme changes, and screenshots are free. The API enforces your plan's allowance and returns `403`/`409` when credits are exhausted.

You don't have to guess the cost — a run reports the actual `creditsCharged` in its terminal object. Check the remaining balance any time:

```bash
curl -s "https://www.daisy.now/api/user/me" -H "Authorization: Bearer $DAISY_API_KEY" | jq '{plan, credits}'
```

## Idempotency

The write endpoints marked _(idempotent)_ in the reference accept an `Idempotency-Key` header (use a UUID). A retried request with the **same key and body** replays the original response, flagged with `X-Idempotent-Replay: true`, instead of running twice. A different body with the same key is a new slot.

Send the same key across retries — that's the point. Always retry on:

- `429 rate_limit:api` — wait the `Retry-After` seconds.
- `500 internal_error` — exponential backoff (1s, 2s, 4s, …, cap ~30s).
- Network/transport errors.

Never retry other `4xx` — the request itself is wrong.

## Errors

Every error is JSON: `{ "code": "...", "message": "..." }` with an appropriate status.

| Code | Status | When | What to do |
|---|---|---|---|
| `unauthorized:api` | 401 | Missing, malformed, revoked, or unrecognized key | Ask the user to re-create the key in the dashboard |
| `forbidden:api` | 403 | Missing scope, wrong plan, or exhausted credits | Upgrade, fix billing, or mint a key with the right scope |
| `bad_request:api` | 400 | Invalid body or query | Fix the request — don't retry |
| `not_found:api` | 404 | Resource doesn't exist or isn't yours | Don't retry, the ID is wrong |
| `conflict:api` | 409 | Active run on the project, or screenshot of an unrendered screen | Cancel/await the run; wait for screen status `done` |
| `rate_limit:api` | 429 | Rate-limited | Honor the `Retry-After` header |
| `internal_error` | 500 | Unexpected server error | Retry with backoff |

## Rate limits

Rate limits apply to **API keys only** (session traffic is exempt). Responses include `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`. On `429`, the `Retry-After` header gives the exact seconds to wait — honor it.

## Limits to remember

- One active run per project (`409` on a concurrent POST `/runs`).
- `label`: 1-120 chars · `brief`: 20-4000 chars · `message`: 1-4000 chars.
- Batch screen update/delete: 1-500 items per request.
- Only screens with status `done` can be screenshotted (`409` otherwise).

## Anti-patterns

- ❌ Polling a run faster than every ~2-3 seconds — wastes rate budget for no gain.
- ❌ Looping `POST /screens` instead of sending one run `message` — runs batch the planning and parallelize generation.
- ❌ Looking for `previewUrl` / `htmlUrl` — those are gone. Read `screen.html` or render `POST /api/screenshots`.
- ❌ Reaching for `PATCH`/`DELETE /screens/:sid` or `PUT /theme` — removed. Use the batch screen endpoints, runs, or `PATCH /projects/:id`.
- ❌ Hardcoding a base URL from an env var — the domain is always `https://www.daisy.now`.
- ❌ Sending random per-attempt `Idempotency-Key` values — reuse the key across retries.
- ❌ Screenshotting a screen before its status is `done` — you'll get a `409`.
