---
name: joyco-lab
description: "Use this skill whenever the user wants to isolate a piece of code, animation, or 3D experiment into a standalone JOYCO Lab experiment and publish it. Triggers include: \"isolate this\", \"publish this to the lab\", \"create a lab experiment\", \"extract this animation/effect/scene\", \"submit to JOYCO lab\", \"make this an experiment\". Also trigger when the user shows you a self-contained visual effect, animation, 3D scene, canvas sketch, or interactive component and wants to share or showcase it. Even if they don't say \"lab\" explicitly — if there's isolatable creative/technical code and a desire to share it, use this skill."
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# JOYCO Lab — Experiment Isolation & Publication

Automates the full workflow of extracting a self-contained experiment from any codebase, scaffolding it with the right JOYCO template, installing dependencies, deploying to Vercel, and publishing to the JOYCO Lab registry.

---

## Prerequisites checklist

Before starting, confirm these are available in the user's environment:

- `joyco` CLI installed (`joyco --version`)
- `vercel` CLI installed (`vercel --version`)
- GitHub auth configured (used by `joyco lab` to open PRs)
- User is logged into Vercel (`vercel whoami`)

If any are missing, tell the user and stop — the workflow depends on all of them.

---

## Step 1 — Understand the experiment

Read the code the user is pointing at. Identify:

1. **What it does** — one-sentence description (you'll use this for the Lab entry)
2. **What type it is** — use the table below to pick a template
3. **What dependencies it needs** — scan imports, note any that aren't in the template baseline

### Template selection guide

| Signals in the code | Template to use |
|---|---|
| `three`, `@react-three/fiber`, `@react-three/drei`, `WebGLRenderer`, `OrbitControls`, canvas 3D | `3d` |
| `gsap`, `ScrollTrigger`, `@studio-freight/lenis`, `framer-motion`, CSS keyframes, SVG animation, canvas 2D drawing | `motion` |

If the experiment mixes both (e.g. GSAP animating a Three.js scene), prefer `3d` and note the GSAP dependency.

If genuinely ambiguous, ask the user before proceeding.

---

## Step 2 — Name the experiment

Derive a human-readable name from what the experiment does. Rules:
- Natural title casing, no slugifying needed (the CLI handles that)
- Short (2–4 words max)
- Descriptive, not generic (❌ `Cool Animation` ✅ `Magnetic Cursor Trail`)

Suggest a name and confirm with the user before running any commands.

---

## Step 3 — Scaffold with JOYCO CLI

```bash
joyco create --template <template-value> <experiment-name>
```

Example:
```bash
joyco create --template motion "Magnetic Cursor Trail"
```

This creates a new directory `<experiment-name>/` with the template scaffolded inside. `cd` into it before the next steps.

---

## Step 4 — Port the experiment code

1. Identify the minimal set of files needed — entry point, shaders, helpers, assets
2. Copy them into the scaffolded project, respecting the template's file structure (typically `src/` or `app/`)
3. Wire the experiment into the template's entry point (usually `src/main.ts`, `src/index.tsx`, or `app/page.tsx` — check the template)
4. Do **not** bring over unrelated code, global state, routing, or backend dependencies

---

## Step 5 — Install missing dependencies

Compare the experiment's imports against what the template already includes.

For each missing package:
```bash
cd <experiment-name> && npm install <package>
```

Then run the dev server to confirm no import errors:
```bash
npm run dev
```

Fix any issues before proceeding. Common ones:
- Missing type packages (`@types/three`, etc.)
- Peer dependency mismatches
- Template assumes a bundler feature the experiment uses differently

---

## Step 6 — Deploy to Vercel

From inside the experiment directory:
```bash
vercel --prod
```

Follow prompts if it's a first-time deploy (link to org, set project name = experiment slug). Copy the production URL — you'll need it for the Lab entry.

---

## Step 7 — Publish to JOYCO Lab

Run the publish command and follow the interactive prompts:
```bash
joyco lab
```

The CLI will ask for:

| Field | What to provide |
|---|---|
| **Name** | Human-readable title (e.g. "Magnetic Cursor Trail") |
| **Description** | One or two sentences about what it is and what's interesting technically |
| **Tags** | Comma-separated (e.g. `gsap, cursor, magnetic, interaction`) — see tag guide below |
| **Preview link** | The Vercel production URL from Step 6 |
| **Preview image** | Optional — URL to an OG/screenshot image if one was generated |

The CLI will open a PR in `joyco-studio/lab` adding the entry to `experiments.json`. Review the PR URL it outputs and share it with the user.

### Tag guide

Use these consistently:

- **By type:** `3d`, `animation`, `interaction`, `shader`, `canvas`, `scroll`, `physics`, `generative`
- **By library:** `three`, `gsap`, `r3f`, `framer-motion`, `lenis`, `matter-js`
- **By visual style:** `typography`, `cursor`, `particles`, `geometry`, `noise`, `fluid`

Pick 3–6 tags. Don't over-tag.

---

## Step 8 — Wrap up

Tell the user:
1. The experiment directory location
2. The live Vercel URL
3. The GitHub PR URL for the Lab registry entry
4. Any follow-up they might want: merging the PR, adding a preview image, promoting on socials

---

## Edge cases

**Experiment uses a private/internal package**: Stop and tell the user. Either the dependency needs to be made public, inlined, or replaced before the experiment can be a standalone public repo.

**Experiment requires a backend/API**: Scope it down to the frontend-only visual part, or ask the user if they want to mock the data. Lab experiments should be fully static/client-side.

**User wants to skip Vercel deploy**: They can provide a different `href` to `joyco lab` — just note that the Hub will render it in an iframe so it must be a publicly accessible URL.

**No clear template match**: Ask the user. Don't guess if it's truly ambiguous — a wrong template wastes setup time.
