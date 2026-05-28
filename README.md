# Daisy — AI Agent Skill

Drive [Daisy](https://daisy.now) — the AI canvas for native mobile app design — directly from your AI coding agent. Ask your agent "design me a settings screen" or "mock up onboarding for my fitness app" and watch screens materialize in your daisy project, with shareable preview URLs ready to drop into a chat or PR.

## What this skill does

- **Plans + generates mobile screens** via Daisy's `/api/v1/projects/:id/runs` orchestrator
- **Iterates on individual screens** (refinement or targeted edits to a specific element)
- **Manages projects** (create, list, archive, delete)
- **Generates and tunes themes** so screens look consistent
- **Returns HMAC-signed preview URLs** (24h TTL) that anyone can open in a browser — no Daisy account needed

The agent always tells you the credit cost up front. Defaults to async runs (`wait="none"`) so long generations don't block your terminal.

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

1. A Daisy **Pro** or **Max** subscription (https://daisy.now/pricing). Free/Starter/Plus tiers can't call the API.
2. An API key from **Settings → API keys** in the daisy dashboard.
3. Two environment variables:

   ```bash
   export DAISY_API_KEY="dsy_live_..."
   # Optional, defaults to https://daisy.now
   export DAISY_BASE_URL="https://daisy.now"
   ```

   Add them to your shell profile so the agent picks them up automatically.

## Things to ask your agent

After installing, try any of these:

- *"Design onboarding for a meditation app: welcome, breathing-style picker, and notifications opt-in."*
- *"Add a profile screen to the Runr project. Avatar, stats grid, and an edit button."*
- *"On the settings screen, change the primary button to red and make the destructive action stand out more."*
- *"Generate a darker theme for this project and regenerate the home feed."*
- *"What did I create on Daisy this week? Show me with preview links."*
- *"Cancel the run that's been going for 8 minutes."*

The agent will pick the right endpoints, batch work into runs when sensible, hand you back clickable preview URLs, and warn you if a step would exceed your remaining credits.

## How it works

This repo contains a single file: `SKILL.md`. It's a Markdown document with YAML frontmatter that teaches your agent:

- Which Daisy API endpoints exist and what they do
- When to use the high-level **runs** orchestrator versus **direct tool** calls
- How to authenticate, handle errors, and respect rate limits
- Credit costs per operation so the agent can be honest about budget
- The full request/response shape for every endpoint, with copy-pasteable `curl` examples

Your agent reads `SKILL.md` once at conversation start (or when invoked by a triggering prompt) and uses it as its working manual.

## Security notes

- API keys are scoped (Full access / Read only / Custom). The agent will respect whatever scopes you grant — pick **Read only** if you want it to inspect but never spend credits.
- Preview URLs are HMAC-signed and expire after 24 hours. Sharing one gives the recipient read access to that single screen, nothing else.
- The skill never asks the agent to store your API key — it always reads from `DAISY_API_KEY`. If your agent tries to print or persist the key, that's a bug in the agent, not the skill.

## Limits

- 1 active run per project (409 on concurrent starts)
- 100 requests / minute per key (429 with `Retry-After` header)
- 5000 requests / hour per user across all keys
- Max 10 API keys per user
- Run timeout: 700 seconds
- Preview URL TTL: 24 hours

## Links

- Daisy: https://daisy.now
- Pricing: https://daisy.now/pricing
- Dashboard: https://daisy.now/dashboard
- Issues with the skill: open one in this repo
- Issues with the API itself: support at daisy.now

## License

MIT. Use it, fork it, embed it in your own agent runtime.
