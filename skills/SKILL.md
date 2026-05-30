---
name: design-mobile-apps-with-daisy
description: Design native mobile app screens via the Daisy REST API. Use this when a user asks you to mock up, design, prototype, or iterate on mobile app screens — onboarding flows, settings pages, home feeds, dashboards, checkout, sign-up, etc. Drives the daisy.so canvas via bearer-token HTTPS, returns signed preview URLs the user can open in a browser.
---

# Design mobile app screens with Daisy

Daisy (https://daisy.now) is an AI canvas for designing native mobile app screens. This skill drives Daisy's public REST API at `/api` so you can spin up projects, generate screens, iterate on them, and hand the user a clickable preview URL — all from inside your coding agent.

## When to use

Use this skill when the user says things like:

- "Design me a settings screen for X"
- "Mock up an onboarding flow for my SaaS"
- "Build a home feed for a fitness app"
- "Iterate on this screen — make the CTA bigger"
- "Sketch the empty state for the inbox"
- "Add a checkout screen to the project we made yesterday"

Do NOT use this skill for: web pages (Daisy is mobile-only), design system tokens divorced from screens, copywriting that doesn't ship pixels, or anything that requires PNG/PDF export (the API returns signed HTML preview URLs only).

## Setup

The user must have a **Pro or Max** Daisy subscription and have created an API key at https://daisy.now/dashboard → API keys.

Read the key from the environment:

```bash
export DAISY_API_KEY="dsy_live_..."
export DAISY_BASE_URL="https://daisy.now"   # override only for self-host
```

All requests use:

```
Authorization: Bearer $DAISY_API_KEY
Content-Type: application/json
```

If `DAISY_API_KEY` is missing or starts with `dsy_test_`, fail early and ask the user to paste the key. Never invent a key.

## Workflow A — Runs (recommended)

Use this when the user describes the work in natural language and you want Daisy to plan + generate everything in one call. **This is the default — prefer it over Workflow B.**

### 1. Make sure you have a project

```bash
# Reuse an existing project if the user already mentioned one
curl -s "$DAISY_BASE_URL/api/projects" \
  -H "Authorization: Bearer $DAISY_API_KEY" | jq '.data[] | {id, name}'

# Or create a new one
curl -s -X POST "$DAISY_BASE_URL/api/projects" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"idea":"fitness tracker for runners","name":"Runr"}'
# → { "id": "abc123", "name": "Runr", ... }
```

### 2. Fire a run

```bash
curl -s -X POST "$DAISY_BASE_URL/api/projects/abc123/runs" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "message": "Design onboarding (3 screens: welcome, goal-picker, permissions) and a home feed with today'\''s run summary.",
    "wait": "none"
  }'
# → 202 { "runId": "run_xyz", "status": "queued", "pollUrl": "/api/runs/run_xyz" }
```

`wait` values:

- `"none"` (default) — returns `202` immediately with `runId` + `pollUrl`. **Use this**; poll every 3-5s.
- `"block"` — blocks up to 700s and returns the final run view. Only use when the user is paying close attention to your terminal and the run is small (1-2 screens).

`tools` (optional) — `{ "mcp": false, "skills": false }` to disable the user's MCP servers / agent skills for this run. Default: both enabled.

**Concurrency**: only one active run per project. A second concurrent POST returns `409 conflict:api`. Either wait for the first run, or cancel it with `DELETE /api/runs/{runId}`.

### 3. Poll until terminal

```bash
while true; do
  STATE=$(curl -s "$DAISY_BASE_URL/api/runs/run_xyz" \
    -H "Authorization: Bearer $DAISY_API_KEY")
  STATUS=$(echo "$STATE" | jq -r .status)
  case "$STATUS" in
    succeeded|failed|cancelled) echo "$STATE" | jq .; break ;;
  esac
  sleep 3
done
```

A final run view looks like:

```json
{
  "id": "run_xyz",
  "status": "succeeded",
  "operations": [
    { "type": "screen_created", "screenId": "scr_1", "label": "Welcome", "ok": true },
    { "type": "screen_created", "screenId": "scr_2", "label": "Goal picker", "ok": true }
  ],
  "output": {
    "summary": "2 screens created",
    "lastAssistantMessage": "Done. Onboarding starts on the welcome screen…"
  },
  "creditsCharged": 20,
  "startedAt": 1735…, "finishedAt": 1735…
}
```

### 4. Resolve screen IDs into preview URLs

The run doesn't return preview URLs directly — fetch each screen:

```bash
for SID in $(echo "$STATE" | jq -r '.operations[].screenId'); do
  curl -s "$DAISY_BASE_URL/api/projects/abc123/screens/$SID" \
    -H "Authorization: Bearer $DAISY_API_KEY" \
    | jq '{label, previewUrl, htmlUrl, status}'
done
```

Or list all screens in the project:

```bash
curl -s "$DAISY_BASE_URL/api/projects/abc123/screens" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  | jq '.data[] | {id, label, previewUrl, status}'
```

### 5. Show the previews to the user

`previewUrl` is HMAC-signed, valid for 24h, requires **no auth**, and renders the screen full-page in mobile dimensions. Print it as a clickable link:

```
✓ Welcome screen      → https://daisy.now/api/preview/eyJhb…
✓ Goal picker         → https://daisy.now/api/preview/eyJhb…
✓ Permissions prompt  → https://daisy.now/api/preview/eyJhb…
```

`htmlUrl` returns the raw HTML (same token + `?raw=1`) — use when the user wants to download or inspect markup.

### 6. Cancel a run (if needed)

```bash
curl -s -X DELETE "$DAISY_BASE_URL/api/runs/run_xyz" \
  -H "Authorization: Bearer $DAISY_API_KEY"
# Queued → cancels immediately, returns 200
# Running → returns 202 with status "aborting"; poll until "cancelled"
# Already terminal → returns the existing terminal view
```

## Workflow B — Direct tools (advanced)

Skip the run orchestrator when you want precise control — e.g. iterating on one screen, or generating screens in a specific order that the AI wouldn't pick. Direct calls are synchronous and return the screen immediately.

### Create one screen

```bash
curl -s -X POST "$DAISY_BASE_URL/api/projects/abc123/screens" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "label": "Settings",
    "brief": "iOS-style settings list: account section with avatar, then notifications, privacy, appearance (light/dark/auto), and a destructive Delete account at the bottom."
  }'
# → 201 with the full screen view, X-Credits-Charged: 10
```

`label` is short (≤120 chars, what the user sees). `brief` is the generation instruction (20-4000 chars; longer + more specific = better output).

### Edit a screen

Two modes:

**Full refinement** (re-generates the whole screen):

```bash
curl -s -X PATCH "$DAISY_BASE_URL/api/projects/abc123/screens/scr_1" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"refinement":"Make the primary CTA orange and double its size. Drop the secondary link."}'
# X-Credits-Charged: 5
```

**Targeted subtree edit** (cheaper, just the one element):

```bash
# Get the screen's HTML first to find data-aid="..." on the element to change
curl -s "$DAISY_BASE_URL/api/projects/abc123/screens/scr_1?raw=1" \
  -H "Authorization: Bearer $DAISY_API_KEY"
# ...find <button data-aid="btn_a3">Continue</button>...

curl -s -X PATCH "$DAISY_BASE_URL/api/projects/abc123/screens/scr_1" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"targetAid":"btn_a3","intent":"Change copy to \"Get started\" and make it full-width."}'
```

### Set or generate the theme

```bash
# Generate a theme from the project idea
curl -s -X POST "$DAISY_BASE_URL/api/projects/abc123/theme/generate" \
  -H "Authorization: Bearer $DAISY_API_KEY"

# Or set one explicitly
curl -s -X PUT "$DAISY_BASE_URL/api/projects/abc123/theme" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"colors":{"primary":"#0A84FF","background":"#0B0F17","text":"#F2F4F7"},"radius":"medium"}'
```

Always set/generate the theme **before** creating screens — every screen inherits from it.

## Showing screens in YOUR UI

If you're building a UI that wraps this skill (e.g. a desktop app, a web frontend), embed previews directly:

```html
<iframe src="{previewUrl}" width="390" height="844" frameborder="0"></iframe>
```

The preview route sets permissive `X-Frame-Options` for embedding under daisy.so's origin. Cross-origin embedding may be blocked depending on the user's plan — fall back to opening `previewUrl` in a new tab.

## Endpoint reference

All paths are under `$DAISY_BASE_URL/api`. Replace `:id` with project ID, `:sid` with screen ID, `:runId` with run ID.

| Method | Path | Scope | Credits |
|---|---|---|---|
| GET | `/me` | implicit | 0 |
| GET | `/projects` | `projects.read` | 0 |
| POST | `/projects` | `projects.write` | 0 |
| GET | `/projects/:id` | `projects.read` | 0 |
| PATCH | `/projects/:id` | `projects.write` | 0 |
| DELETE | `/projects/:id` | `projects.write` | 0 |
| GET | `/projects/:id/screens` | `screens.read` | 0 |
| POST | `/projects/:id/screens` | `screens.write` | **10** |
| GET | `/projects/:id/screens/:sid` | `screens.read` | 0 |
| PATCH | `/projects/:id/screens/:sid` | `screens.write` | **5** |
| DELETE | `/projects/:id/screens/:sid` | `screens.write` | 0 |
| PUT | `/projects/:id/theme` | `themes.write` | 0 |
| PUT | `/projects/:id/design-contract` | `themes.write` | 0 |
| POST | `/projects/:id/runs` | `runs.write` | variable |
| GET | `/projects/:id/runs` | `runs.read` | 0 |
| GET | `/runs/:runId` | `runs.read` | 0 |
| DELETE | `/runs/:runId` | `runs.write` | 0 |
| GET | `/usage?days=&projectId=` | `usage.read` | 0 |
| GET | `/preview/:token` | (signed) | 0 |

## Credit costs

- **Screen create**: 10 credits (≈$0.10)
- **Screen edit**: 5 credits
- **Run**: sum of all screens created / edited inside the run
- Everything else (reads, theme changes, deletes, project ops): **free**

Free-tier and Plus accounts can't call this API at all — they get `403 forbidden:api`. If you see that, tell the user to upgrade at https://daisy.now/pricing.

Check remaining credits before a big run:

```bash
curl -s "$DAISY_BASE_URL/api/me" -H "Authorization: Bearer $DAISY_API_KEY" \
  | jq '{credits, creditsAllowance}'
```

If `credits` is `"unlimited"`, the user is an admin and you can ignore the budget.

## Idempotency

Send `Idempotency-Key: <uuid>` on every write (`POST`/`PATCH`/`PUT`). Daisy returns the cached response (with `X-Idempotent-Replay: true`) for 24 hours if you retry the same key + same body. Different body = different slot, even with the same key.

Always retry on:

- `429 rate_limit:api` (wait `Retry-After` seconds)
- `503 offline:api` (exponential backoff: 1s, 2s, 4s, …, cap at 30s, give up after ~3 minutes)
- Network/transport errors

Never retry on `4xx` other than 429 — the request itself is wrong.

## Error shape

Every error is JSON:

```json
{ "code": "forbidden:api", "message": "…", "cause": "Public API access requires the Pro or Max plan." }
```

Common codes:

| Code | When | What to do |
|---|---|---|
| `unauthorized:api` | bad/missing/revoked key | Ask user to re-create the key in the dashboard |
| `forbidden:api` | no scope, wrong tier, or `subscriptionStatus` is not `active`/`trialing` | Tell user to upgrade or fix billing |
| `not_found:api` | project/screen/run doesn't exist | Don't retry, the ID is wrong |
| `conflict:api` | active run on project, or `Idempotency-Key` collision with different body | For active run: cancel or wait; for idempotency: use a new key |
| `rate_limit:api` | rate-limited | Honor `Retry-After` header |
| `bad_request:api` | invalid body | Fix the request — don't retry |
| `offline:api` | transient backend issue | Retry with backoff |

## Rate limits

Per key: 100 req/min, 1000 req/hour. Per user: 5000 req/hour across all keys.

429 responses include:

```
Retry-After: 12
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1735…
```

Honor `Retry-After` — Daisy uses fixed windows, not sliding, so the value is exact.

## Limits to remember

- One active run per project (409 on concurrent POST `/runs`)
- Max 10 API keys per user
- `brief` field: 20-4000 chars; `refinement`: 1-800 chars; `message`: 1-4000 chars
- Run max duration: 700s
- Preview URL TTL: 24h (just refetch the screen to get a fresh URL)

## Anti-patterns

- ❌ Polling `/runs/:runId` faster than every 2 seconds — wastes rate budget for no gain
- ❌ Calling `POST /screens` in a tight loop instead of using `POST /runs` with one `message` — runs batch the planning + parallelize generation under one credit charge
- ❌ Treating `previewUrl` as auth-gated — it's an unauthenticated signed URL safe to print to terminal
- ❌ Sending `Idempotency-Key` with random per-attempt values — the whole point is reusing the key across retries
- ❌ Reading `data-aid` from your own guess of the HTML — always fetch with `?raw=1` first; aids are server-stamped and unstable across regenerations
