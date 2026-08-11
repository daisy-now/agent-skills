# Writing briefs that come out right

Read [../SKILL.md](../SKILL.md) first. This is the craft of writing the run `message` (and the `brief` for a direct single-screen create). Daisy designs from your words — a specific, concrete brief comes back as good screens; a vague one comes back as a guess. Getting it right the first time also saves the user a whole edit cycle (and the credits that cycle would charge).

## The one idea

Think of the `message` as a **design spec, not a wish**. Daisy fills in the standard mobile screen grammar (navigation, lists, forms, CTAs) on its own — your job is to pin down everything that is specific about this app.

- **Name the product and the audience.** *"Onboarding for a meditation app aimed at beginners"* sets the tone and complexity. A bare *"onboarding"* leaves it guessing.
- **Enumerate the screens.** One line per screen, in flow order, with what is on it and what it does. A flow Daisy can see (welcome → goal-picker → permissions) beats a lump (just *"onboarding"*). Name the screens the flow implies but the user didn't list — a profile screen implies an edit screen.
- **Say what makes this app different.** The layout grammar is the same everywhere; the differentiator is your copy, your palette, and the one interaction worth remembering.
- **Give it real content.** Real copy, real numbers, real product details — not "Lorem ipsum" placeholders. A screen that looks like the real thing reads like the real thing.
- **Say what to avoid.** One or two lines of negative guidance ("no sign-up wall", "keep it to a single stats card") prevents the predictable failure modes.

## Anatomy of a strong run message

Five parts, roughly in order:

1. **The product in one sentence.** What the app is and who it is for.
2. **The screens, enumerated.** One line each, in the order the user will see them.
3. **Platform conventions.** There is no platform switch in the API — Daisy reads it from your words, so say it in natural language: *"iOS-style, bottom tab bar"* or *"Android, Material 3, FAB for the primary action."* Skip this when the user doesn't care.
4. **Look and feel.** Tone, color direction, density. If the project has a theme, the screens already inherit it (set the theme first — see [direct-ops.md](direct-ops.md)) — then spend the message on the screens, not the palette.
5. **Explicitly what not to do.** One or two constraints, stated as things to avoid.

### Weak

> Design a workout app.

That is a mood, not a spec — expect generic fitness screens.

### Strong

> Design the core flow for "Pace", a running app for beginners. 3 screens:
>
> 1. Home — today's run summary: distance, pace, and a big start button, plus a 7-day streak card.
> 2. Start run — full-screen pre-run state: GPS status, a big "Start" button, and last run's PR to beat.
> 3. Post-run — completion state: time, distance, splits, and a "Share" call to action.
>
> iOS-style, light theme, one accent color, friendly but not playful. Keep every screen to a single primary action. No login, no sign-up, no ads.

Every screen is described by what is on it and what it is for, platform and tone are pinned down, and the anti-patterns are named. `message` is 1–4000 chars — a brief like this fits comfortably.

## Writing screen-by-screen briefs

The direct create path (see [direct-ops.md](direct-ops.md)) takes a `label` and a `brief` (20–4000 chars) per screen, synchronously. Same craft, tighter:

- **`label`** is the short name the user sees — specific ("Home", "Post-run summary"), not generic ("Screen 1").
- **`brief`** is the design instruction: what the screen is for, its elements top to bottom, and any state (empty, loading, done).

> "Settings for a sleep-tracking app: account section with avatar and name, then notification toggles (bedtime reminders, weekly summary), then a dark-mode switch, and a red Delete account at the bottom. iOS-style grouped list."

The more specific you are about elements and their order, the closer the first pass is to the target.

## Iterating: turning feedback into an edit message

The first run is a draft. When the user reacts, convert the feedback into a **targeted edit message**: name the screen, the element, and the change. Vague feedback comes back vague.

- "make the button pop" → *"On the Start run screen, make the Start button the accent color and full-width, and add 'Ready when you are' above it."*
- "home is too busy" → *"On Home, remove the stats grid; keep the streak card and the big start button, and make it feel calmer."*

Naming the screen matters — runs edit existing screens in place, so a message that says *"the home screen"* lands precisely, and one that doesn't leaves the run to guess which screen you mean.

One rich message per iteration beats several small ones: runs plan and parallelize, and each run charges credits for the screens it creates or edits — report the real `creditsCharged` back so the user knows the spend.

## Checklist

Before you fire the run, the `message` should:

- [ ] Name the product and audience
- [ ] Enumerate each screen with what is on it and what it does
- [ ] Pin the platform and look if the user cares (theme set first — screens inherit it)
- [ ] Use real copy and realistic data
- [ ] Name 1–2 things to avoid
- [ ] Fit in 4000 chars (it will — good briefs are a few short paragraphs)

If you can't tick all six, you are asking Daisy to guess. Spend one more message on the brief — it is the cheapest edit you will make.
