# Direct screen ops (advanced)

Read [../SKILL.md](../SKILL.md) first. **Runs are the default** — reach here only when you want precise, synchronous control instead of letting a run plan the work: generating one screen, repositioning the canvas, pushing HTML you authored yourself, setting the theme, or embedding screens in your own UI.

## Contents

- Create one screen
- Edit a screen
- Batch update screens
- Batch delete screens
- Set the theme
- Embed screens in your own UI

## Create one screen

Synchronous. Consumes credits.

```bash
curl -s -X POST "https://www.daisy.now/api/projects/abc123/screens" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "label": "Settings",
    "brief": "iOS-style settings list: account section with avatar, then notifications, privacy, appearance (light/dark/auto), and a destructive Delete account at the bottom."
  }'
# → 201 with the created screen, including `html`.
```

`label` is the short name the user sees (1–120 chars). `brief` is the generation instruction (20–4000 chars; longer and more specific = better output).

## Edit a screen

There is **no per-screen AI edit endpoint**. Two ways to change a screen:

- **Natural language (preferred)** — fire a run (see SKILL.md) with a message like *"On Settings, make the primary CTA orange and double its size."* The run edits the existing screen in place.
- **Direct HTML** — if you already have the new markup, push it through the batch update below.

## Batch update screens

Field-level updates to many screens at once — canvas drag/resize, inline HTML edits. Free.

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

Each item: `{ id, x?, y?, height?, userResized?, html?, prompt?, generationPrompt?, status? }` (1–500 items).

## Batch delete screens

```bash
curl -s -X DELETE "https://www.daisy.now/api/projects/abc123/screens" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"ids":["scr_1","scr_2"]}'
# → { "ok": true }  (1–500 ids)
```

## Set the theme

The theme lives on the project; there is no separate theme route. Set it **before** generating — every screen inherits it. The current theme ships in `GET /projects/:id`.

```bash
curl -s -X PATCH "https://www.daisy.now/api/projects/abc123" \
  -H "Authorization: Bearer $DAISY_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"theme":{"colors":{"primary":"#0A84FF","background":"#0B0F17","text":"#F2F4F7"},"radius":"medium"}}'
# → the updated full project
```

## Embed screens in your own UI

There are no public preview URLs. Two ways to show a screen in a desktop or web frontend:

- **Inline HTML** with an `srcdoc` iframe:

  ```html
  <iframe srcdoc="{escaped screen.html}" width="390" height="844" frameborder="0"></iframe>
  ```

- **Image bytes** from `POST /api/screenshots` — display as a saved file or `<img src="data:image/png;base64,…">`.
