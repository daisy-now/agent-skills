# Daisy AI Agents Skill

Drive [Daisy](https://www.daisy.now) directly from your AI coding agent. Ask your agent "design me a settings screen" or "mock up onboarding for my fitness app" and watch screens materialize in your daisy project — the agent reads back each screen's rendered HTML and can render them to PNG/JPEG images to drop into a chat or PR.

## Installation

Browse and install interactively:

```bash
npx skills add daisy-now/agent-skills
```

## What this skill does

- **Plans + generates mobile screens** via Daisy's `/api/projects/:id/runs` orchestrator
- **Iterates on screens** by sending natural-language runs, or pushes direct HTML edits through the batch screen endpoint
- **Manages projects** (create, list, update, delete)
- **Tunes themes** (via `PATCH /api/projects/:id`) so screens look consistent
- **Reads rendered HTML** straight off each screen and **renders PNG/JPEG images** via `POST /api/screenshots`

The agent reports the credit cost from each run's `creditsCharged`. Defaults to async runs (`wait="none"`) so long generations don't block your terminal.

## Install

### Claude Code

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/<your-org>/daisy-skill ~/.claude/skills/daisy
```

Claude Code auto-discovers it on next launch. Verify with `/skills` and look for `design-mobile-apps-with-daisy`.

### Cursor / other agent runtimes

Most agent skill catalogs read a `SKILL.md` with YAML frontmatter from a folder. Point your runtime at this repo and follow its docs.

## Prerequisites

1. A Daisy **Pro** or **Max** subscription (https://www.daisy.now/pricing). Free/Plus tiers can't call the API.
2. An API key from **Settings → API keys** in the daisy dashboard.
3. One environment variable:

   ```bash
   export DAISY_API_KEY="dsy_live_..."
   ```

   Add it to your shell profile so the agent picks it up automatically. The domain is always `https://www.daisy.now` — there is no base-URL override.

## Things to ask your agent

After installing, try any of these:

- *"Design onboarding for a meditation app: welcome, breathing-style picker, and notifications opt-in."*
- *"Add a profile screen to the Runr project. Avatar, stats grid, and an edit button."*
- *"On the settings screen, change the primary button to red and make the destructive action stand out more."*
- *"Set a darker theme for this project and regenerate the home feed."*
- *"What did I create on Daisy this week? Render each screen to a PNG."*
- *"Cancel the run that's been going for 8 minutes."*

The agent will pick the right endpoints, batch work into runs when sensible, save the rendered HTML or screenshots locally for you, and report the credits each run charged.

## How it works

The skill is built around **progressive disclosure** — the lean part loads first, the rest only when needed:

```
skills/
├── SKILL.md                  # The one path: run → poll → HTML → image, plus guardrails
└── reference/
    ├── api.md                # Endpoint table, scopes, errors, rate limits, idempotency, credits
    ├── direct-ops.md         # Advanced: single-screen create, batch edits, theme, embedding
    └── briefing.md           # Writing briefs that come out right: run messages & screen briefs
```

At startup your agent loads only the `name` + `description` from the frontmatter. When a prompt matches, it reads `SKILL.md` — the opinionated happy path it follows for ~90% of requests. It opens a `reference/` file **only** when it hits an error or needs the advanced path, so idle token cost stays near zero.

Together these teach your agent:

- The single recommended workflow: the **runs** orchestrator, async, polled
- How to authenticate, handle errors, retry safely, and respect rate limits
- Credit costs per operation so the agent can be honest about budget
- The full request/response shape for every endpoint, with copy-pasteable `curl`
- When to drop down to **direct screen ops** for precise control
- The craft of writing a run `message` — a spec, not a wish — so screens come out right the first time

## Security notes

- API keys are scoped (`full_access` / `read_only` / custom). The agent will respect whatever scopes you grant — pick **read_only** (`projects:read`, `screens:read`, `runs:read`) if you want it to inspect but never spend credits.
- Screens return their rendered HTML directly, and screenshots are fetched with your key — there are no public preview URLs to leak.
- The skill never asks the agent to store your API key — it always reads from `DAISY_API_KEY`. If your agent tries to print or persist the key, that's a bug in the agent, not the skill.

## Limits

- 1 active run per project (409 on concurrent starts)
- Per-key rate limits (429 with `Retry-After`; see the `X-RateLimit-*` response headers)
- Batch screen update/delete: 1–500 items per request
- Only screens with status `done` can be rendered to an image (409 otherwise)

## Links

- Daisy: https://www.daisy.now
- Pricing: https://www.daisy.now/pricing
- Dashboard: https://www.daisy.now/dashboard
- Issues with the skill: open one in this repo
- Issues with the API itself: support at daisy.now

## License

MIT. Use it, fork it, embed it in your own agent runtime.
