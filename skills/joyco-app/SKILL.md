---
name: joyco-app
description: >
  Carry a new JOYCO internal product app from plan to working build. Locks a
  plan first (one the dev already refined this session, brought from another
  agent session, or one you draft together), then runs the opinionated kickoff
  — Next.js + TypeScript + Tailwind wired to the JOYCO UI shadcn registry
  (@joyco/ui kit: components, theme, fonts from hub.joyco.studio), JOYCO lint
  config, and the Claude Code agent — and then builds the app out to the plan.
  User-invocable as /joyco-app <project-name>. Triggers on "new joyco app",
  "start a joyco project", "kickoff an internal joyco app", "build this as a
  joyco app", "set up joyco/ui", and on mentions of the joyco CLI, the @joyco
  namespace, or hub.joyco.studio. Do NOT use this to publish a standalone
  experiment (that's joyco-lab) or to scaffold a 3d/motion/library starter via
  `joyco create` — those are experiment/library templates, not product apps.
license: MIT
user-invocable: true
argument-hint: <project-name>
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# JOYCO App — Plan → Kickoff → Build

This skill carries a new JOYCO internal **product** app through the whole loop: it locks a plan, runs a consistent and opinionated kickoff, then builds the app out to that plan. It is **not** a scaffolder that stops at `pnpm dev` — the kickoff is the shared, non-negotiable starting point; the plan is what the project becomes.

Three phases, always in order:

1. **Lock the plan** — never scaffold without a confirmed plan.
2. **Kickoff** — identical every time: Next.js + JOYCO UI kit + lint + agent.
3. **Build to the plan** — translate the plan into routes/components and implement until it's done.

---

## When to use

- The dev wants to start a new internal JOYCO product / dashboard / marketing site and carry it forward, not just generate a skeleton.
- The dev says "new joyco app", "kickoff a project", "build this as a joyco app", "set up joyco/ui".

Do **not** use this skill for:
- **Publishing a self-contained experiment** → that's the `joyco-lab` skill.
- **Scaffolding a 3D / motion / TS-library starter** → that's `joyco create -t 3d|motion|library` (experiment/library templates, not product apps).

---

## Mental model

Two ideas:

1. **The kickoff is opinionated and fixed.** Every JOYCO app starts the same way — same stack, same registry wiring, always the `@joyco/ui` kit. This consistency is the point; don't improvise it per project.
2. **The plan is the variable.** What gets built on top — routes, screens, which extra `@joyco/*` components, data/auth — comes entirely from the plan. The skill follows the plan; it doesn't invent scope.

There is **no `joyco create` template for product apps**. A JOYCO app is a plain Next.js app plus the JOYCO shadcn registry (`https://hub.joyco.studio/r/{name}.json`, namespaced `@joyco`) layered on top. The `joyco` CLI wrappers (`components`, `agents`, `lint-config`) are mostly *interactive TUIs* — as an agent, prefer the non-interactive command each runs under the hood (given at every step).

```
LOCK PLAN  →  [ create-next-app → shadcn init → register @joyco → @joyco/ui kit → lint → agent ]  →  BUILD TO PLAN
             └──────────────────────── opinionated, identical every run ────────────────────────┘
```

---

## Prerequisites

Confirm before starting (don't install global tooling without asking):

- Node ≥ 18 and `pnpm` (the JOYCO default package manager).
- `joyco` CLI available (`joyco --version`). Not strictly required — every step has a raw `pnpm dlx` equivalent — but it's the house tool. Install with `pnpm add -g joyco` if missing and the user wants it.

---

# Phase 1 — Lock the plan

**No commands run until there's a confirmed plan.** This holds regardless of app size — even a "simple" app gets a one-paragraph plan so the build has a target.

### 1a. Find an existing plan first

The dev may already have a plan. Look before asking — in priority order:

1. **This session** — the dev refined a plan or spec in the current conversation. Use it.
2. **A plan the dev points to** — a `PLAN.md` / spec / PRD in the repo or cwd, a pasted plan from another agent session, a linked issue/ticket. Read it.
3. **Plan mode output** — if planning happened in plan mode before the skill fired.

If you find one, **restate it back concisely** (purpose, routes, key components/flows) and confirm it's the source of truth before continuing. Don't silently re-plan something the dev already settled.

### 1b. If there's no plan, draft one together

Gather just enough to build against, then write it down:

- **Purpose** — what the app is, one or two sentences.
- **Routes / screens** — the pages and their rough hierarchy.
- **Key flows & interactions** — anything beyond static pages (auth, forms, realtime, signature JOYCO motion/media moments).
- **Data & external deps** — APIs, DB, auth provider, env needs.
- **Milestones** — an ordered list of build chunks so Phase 3 has a sequence.

Keep it short and concrete. Confirm it with the dev before moving on.

### 1c. Gate

Do not proceed to Phase 2 until the dev has confirmed the plan. The plan drives component selection and the build order in Phase 3 — treat it as the contract for the rest of the loop.

---

# Phase 2 — Kickoff (opinionated, identical every run)

This sequence is the same for every JOYCO app. Don't vary it by project. Ask for the project name if not given (lowercase, hyphens), then run it top to bottom.

## Step 1 — Scaffold the Next.js app

```bash
pnpm create next-app@latest <project-name> \
  --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-pnpm
```

`cd <project-name>` before every following step. Keep `--tailwind` and `--app` — the JOYCO registry assumes Tailwind and the App Router. Honor other strong preferences (no `src/`, different alias) if the dev has them.

## Step 2 — Initialize shadcn

```bash
pnpm dlx shadcn@latest init -d
```

`-d` accepts defaults (neutral base color, CSS variables). Creates `components.json` and wires the Tailwind tokens / `cn` helper. The base color barely matters — the JOYCO theme (Step 4) overrides the tokens.

## Step 3 — Register the `@joyco` namespace (the load-bearing step)

`shadcn add @joyco/<name>` fails unless `components.json` knows where `@joyco` resolves. Add a `registries` block to the existing `components.json` (don't clobber the generated file):

```json
{
  "registries": {
    "@joyco": "https://hub.joyco.studio/r/{name}.json"
  }
}
```

Verify it resolves:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://hub.joyco.studio/r/button.json   # expect 200
```

## Step 4 — Install the JOYCO UI kit (always)

The `@joyco/ui` umbrella style is the kickstart, every time. One add pulls the *full* kit — all base components, brand color tokens (theme), and font config — so the app is on-brand before any product code.

```bash
pnpm dlx shadcn@latest add @joyco/ui
```

Unconditional — there's no "lean start" variant. It includes:

- **Components** — base shadcn-style primitives (button, input, select, card, …), JOYCO-themed.
- **Theme** (`@joyco/joyco-theme`) — light/dark color tokens in your global CSS.
- **Fonts** (`@joyco/joyco-fonts`) — adds `src/lib/fonts.ts` (Public Sans + Roboto Mono).

Then **wire the fonts up** — the kit only writes `fonts.ts`, it doesn't apply it. Import the exported font variables in `src/app/layout.tsx` and put their classNames on `<html>`/`<body>` (open `fonts.ts` to confirm the export names).

## Step 5 — Lint + format config

```bash
joyco lint-config
```

Downloads `eslint.config.mjs`, `.prettierrc`, `.prettierignore` from `joyco-studio/lint-config` and installs the dev deps (eslint, eslint-config-next, eslint-plugin-import-x, eslint-plugin-simple-import-sort, prettier, prettier-plugin-tailwindcss). Prompts only if those files already exist (create-next-app's `--eslint` writes one) — confirm the overwrite. No CLI? Clone those files manually and `pnpm add -D` the deps.

## Step 6 — Add the Claude Code agent

```bash
pnpx @joycostudio/scripts@latest agents -s claude
```

Exactly what `joyco agents` → "Claude Code" runs, but non-interactive (`-s claude`). Drops the JOYCO agent config into the project. (Swap `-s cursor` / `-s codex` for a different platform.)

## Step 7 — Verify the kickoff

```bash
pnpm dev
```

Confirm the dev server boots, JOYCO fonts render, theme tokens apply (check a themed component like `@joyco/button`), and there are no missing-import errors. Fix before moving to Phase 3.

---

# Phase 3 — Build to the plan

The kickoff is on-brand but empty. Now execute the plan — this is where the project actually takes shape, and where the skill earns "start to end of the loop."

### 3a. Translate the plan into structure

Map the plan's routes/screens onto the App Router (`src/app/...`), set up shared layout, navigation, and any providers (auth, data, theme toggle) the plan calls for.

### 3b. Pull the extra components each screen needs

`@joyco/ui` already gave you the base primitives. The **whole registry is available on top** — JOYCO signature components (`marquee`, `magnetic`, `scramble`, `infinite-list`, media/canvas pieces, …) and other extras. Pull them as the plan's screens require — **don't work from memory, the registry changes.** Fetch the live catalog:

```bash
curl -s https://hub.joyco.studio/r/registry.json | \
  python3 -c "import sys,json; [print(f\"{i['name']:24} {i.get('description','')}\") for i in json.load(sys.stdin)['items']]"
```

Add the slugs you need by name (registry/peer deps install automatically):

```bash
pnpm dlx shadcn@latest add @joyco/marquee @joyco/magnetic
```

This is the non-interactive equivalent of `joyco components` (avoid that interactive picker in an agent flow).

### 3c. Implement, milestone by milestone

Work the plan's milestones in order. Keep `pnpm dev` running, verify each chunk in the browser, and lint as you go. Iterate with the dev until the plan's milestones are met — that's the end of the loop, not the scaffold.

---

## Pitfalls

1. **Scaffolding before the plan is locked.** Phase 1 is a hard gate. Don't run a single command until the plan is confirmed — even for a "simple" app.
2. **Improvising the kickoff.** Phase 2 is fixed and opinionated. Don't swap the stack, skip `@joyco/ui`, or "optimize" the sequence per project — consistency is the value.
3. **Skipping the namespace registration (Step 3).** Every `@joyco/*` add fails with a registry-not-found error until `components.json` has the `registries` block.
4. **Driving the interactive `joyco` TUIs.** `joyco components` and `joyco create` are arrow-key pickers — they hang an agent. Use the `pnpm dlx shadcn add` / `create-next-app` commands directly. `joyco lint-config` and `joyco agents -s claude` are fine.
5. **Adding fonts but never wiring them.** `@joyco/ui` writes `fonts.ts` — nothing renders until you import and apply it in `layout.tsx`.
6. **Dropping Tailwind or the App Router.** The registry components assume both.

---

## Checklist

**Phase 1 — Plan**
- [ ] An existing plan was looked for (this session / a doc the dev points to / plan mode) before drafting a new one.
- [ ] A plan exists, is written down, and the dev confirmed it. No commands ran before this.

**Phase 2 — Kickoff**
- [ ] Next.js app created with TypeScript + Tailwind + App Router.
- [ ] shadcn initialized; `@joyco` namespace registered in `components.json` and a registry URL returns 200.
- [ ] `@joyco/ui` kit installed (components + theme + fonts); fonts imported/applied in `layout.tsx`.
- [ ] Lint config present and dev deps installed; Claude Code agent added.
- [ ] `pnpm dev` boots clean; theme + fonts visibly applied.

**Phase 3 — Build**
- [ ] Plan's routes/screens mapped onto the App Router.
- [ ] Extra `@joyco/*` components pulled from the live registry as screens needed them.
- [ ] Milestones implemented and verified in the browser with the dev.
