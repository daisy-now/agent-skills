---
name: design-mobile-apps-with-daisy
description: Design and iterate on native mobile app screens through the Daisy REST API (daisy.now) — onboarding flows, settings, home feeds, dashboards, checkout, sign-up, profiles, empty states. Use when the user asks to mock up, design, prototype, sketch, generate, or change mobile app screens. Generates screens from a natural-language brief, reads back each screen's rendered HTML, and renders screens to PNG/JPEG. Mobile only — not for web pages.
---

# Design mobile app screens with Daisy

Daisy (https://www.daisy.now) is an AI canvas for native mobile screens. This skill drives its REST API so you generate screens from a brief, read their rendered HTML, and render them to images — all from your agent.

## The one path

For **every** screen request: ensure a project, fire one run, poll until it finishes. Do not hand-build screens one `POST` at a time.

```
1. Ensure a project exists   →  reuse the user's, else create one
2. Fire a run with the brief →  POST /runs, wait="none"
3. Poll until terminal        →  every 3s, until succeeded|failed|cancelled
   then report what was made + creditsCharged, and link the canvas
```

That is the whole job. After a successful run the screens are **live on the user's canvas** at `https://www.daisy.now/project/:id` — you do not have to download, render, or save anything. Fetching the HTML or an image is a separate, optional step you take only when the task calls for it (see [Getting screens out](#getting-screens-out-only-when-needed)).

A **run** is Daisy's orchestrator: from one natural-language `message` it plans the screens, generates them in parallel, and edits existing screens in place. One run covers *"design welcome + goal-picker + permissions onboarding, plus a home feed"* or *"make the CTA on Settings orange and bigger."* Always prefer one rich `message` over many calls.

## Auth — read before any call

- The domain is always `https://www.daisy.now`. No base-URL override, no `/v1` prefix; every route is under `/api`.
- Read the key from `DAISY_API_KEY`. Send it as `Authorization: Bearer $DAISY_API_KEY` (or `x-api-key: $DAISY_API_KEY`). JSON bodies also need `Content-Type: application/json`.
- If `DAISY_API_KEY` is missing or starts with `dsy_test_`, **stop and ask the user for a live `dsy_live_` key.** Never invent one.
- Generating screens requires a **Pro or Max** plan. Free/Plus get `403 forbidden:api` → tell the user to upgrade at https://www.daisy.now/pricing. A read-only key can inspect but not spend credits.

## Steps

### 1 — Project

```bash
# Reuse if the user named one
curl -s "https://www.daisy.now/api/projects" \
  -H "Authorization: Bearer $DAISY_API_KEY" | jq '.data[] | {id, name}'

# Else create one (returns 201 with the full project, including its id)
curl -s -X POST "https://www.daisy.now/api/projects" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"idea":"fitness tracker for runners","name":"Runr"}'
```

Set the theme **before** generating if the user cares about look — every screen inherits it. See [reference/direct-ops.md](reference/direct-ops.md).

### 2 — Fire the run

```bash
curl -s -X POST "https://www.daisy.now/api/projects/abc123/runs" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "message": "Design onboarding — 3 screens: welcome, goal-picker, permissions — and a home feed showing today'\''s run summary.",
    "wait": "none"
  }'
# → 202 { "runId": "run_xyz", "status": "queued", "projectId": "abc123",
#         "pollUrl": "/api/projects/abc123/runs/run_xyz" }
```

A longer, more specific `message` produces better screens. `message` is 1–4000 chars.

### 3 — Poll until terminal

```bash
while true; do
  STATE=$(curl -s "https://www.daisy.now/api/projects/abc123/runs/run_xyz" \
    -H "Authorization: Bearer $DAISY_API_KEY")
  case "$(echo "$STATE" | jq -r .status)" in
    succeeded|failed|cancelled) echo "$STATE" | jq .; break ;;
  esac
  sleep 3
done
```

The terminal Run object reports `creditsCharged` (the real cost), `output.summary`, and the screens it touched in `operations[].screenId`:

```jsonc
{
  "id": "run_xyz", "projectId": "abc123",
  "status": "succeeded",                          // queued|running|succeeded|failed|cancelled
  "input": { "message": "…", "modelKey": null, "enableSkills": true },
  "output": { "summary": "2 screens created", "lastAssistantMessage": "Done…" },
  "operations": [ { "type": "screen_created", "screenId": "scr_1", "label": "Welcome", "ok": true } ],
  "creditsCharged": 20, "error": null
}
```

## Getting screens out (only when needed)

After a successful run the screens already exist on the canvas — that is the deliverable. Reach for these endpoints **only** when the request (or your own verification) actually needs an artifact. Don't fetch or save by default, and never write a file to the user's disk unless they asked for a file. Screen ids come from the run's `operations[].screenId`.

**The user wants an image** — a preview, a screenshot, "show me", something to drop into a chat or PR — or you want to eyeball the result:

```bash
curl -s -X POST "https://www.daisy.now/api/screenshots" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"projectId":"abc123","screenId":"scr_1","format":"png"}' -o welcome.png
# PNG default ("jpeg" too). 409 until the screen's status is "done".
```

**The user wants the markup / code**, or you need to inspect a screen — the single-screen GET carries the rendered `html` (the list endpoint returns metadata only):

```bash
curl -s "https://www.daisy.now/api/projects/abc123/screens/scr_1" \
  -H "Authorization: Bearer $DAISY_API_KEY" | jq '{id, label, status, html}'
# status is loading|done|error — only "done" screens have final html
```

Produce the one form the user asked for — a canvas link, an image, or the code — not all three.

## Cancel a run

```bash
curl -s -X DELETE "https://www.daisy.now/api/projects/abc123/runs/run_xyz" \
  -H "Authorization: Bearer $DAISY_API_KEY"
# Active → 202 status "aborting"; poll until "cancelled". Terminal → returned unchanged.
```

## Rules

- **Poll `wait:"none"` every ~3s.** Use `wait:"block"` only for a 1–2 screen run the user is watching live — it holds the connection open and can run for minutes.
- **One run per project at a time.** A concurrent `POST /runs` returns `409`. Wait for the current run or cancel it.
- **One run, rich message** beats looping `POST /screens` — runs plan and parallelize; direct creation does not.
- **Screenshot only `done` screens** (else `409`).
- **Reuse the `Idempotency-Key` across retries** of the same request — that is the point of it. A fresh key each attempt defeats it.
- **Report `creditsCharged`** from each run so the user knows the spend. Reads, deletes, batch field edits, theme changes, and screenshots are free.
- **Don't materialize what wasn't asked for.** A finished run is the deliverable — link the canvas (`https://www.daisy.now/project/:id`). Fetch HTML or render a PNG only when the user wants that artifact or you need to verify a screen, and write files to disk only when they ask for a file.
- **Never** look for `previewUrl`/`htmlUrl` (gone — read `html`), a `/v1` path, a base-URL env var, a per-screen edit endpoint, or a `tools.mcp` field (only `tools.skills` exists).

## Errors

Errors are JSON `{ "code", "message" }`. Retry only `429` (honor `Retry-After`), `500` (exponential backoff), and transport errors — reusing the same `Idempotency-Key`. Never retry other `4xx`; the request is wrong. Full code table, scopes, rate limits, and idempotency rules: [reference/api.md](reference/api.md).

## Beyond the happy path

- **Precise control** — generate a single screen, drag/resize on the canvas, push your own HTML, set the theme, or embed screens in your own UI → [reference/direct-ops.md](reference/direct-ops.md)
- **Full endpoint table, scopes & plans, error codes, rate limits, idempotency, credits** → [reference/api.md](reference/api.md)
