---
name: joyco-ui
description: >
  Use the JOYCO UI kit correctly when building interfaces in a JOYCO project —
  a bento/console-style component library on shadcn/ui, Radix, and Tailwind,
  served from the JOYCO hub. Trigger when building or editing UI in a JOYCO
  app, when imports come from `@/components/ui/*`, when the user installs
  `@joyco/ui` or `@joyco/<component>` via shadcn, or mentions Cluster, Filler,
  the bento layout, or the JOYCO design system. This skill does NOT describe
  the components inline — it routes you to the live docs and source on the hub
  so the guidance never goes stale. Pair with `joyco-app` for new-project
  setup.
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# JOYCO UI Kit

A bento/console-style component library built on shadcn/ui with Radix primitives and Tailwind, served from the JOYCO hub (`hub.joyco.studio`). Components are installed into the project at `@/components/ui/*` via the `@joyco` shadcn registry.

**This skill deliberately holds no component API.** Variants, props, and behavior live in the hub and in the installed source — both of which change. Memorizing them here guarantees drift. The skill's only job is to send you to the live source and keep you from improvising the kit's conventions.

> Setting up a *new* project (Next.js + the kit + lint + agent)? That's the `joyco-app` skill. This skill is about *using* the kit once it's installed.

---

## The one rule: don't work from memory

The kit's API changes. Before using a component, read its live definition. There are three sources, in order of what you need:

1. **The Cluster / Filler layout primitives** — the heart of the system. Read the live doc:
   `https://hub.joyco.studio/components/cluster.md`
   It covers the transparent-container concept, the `bg` cascade, `Filler`, `asChild`/inline use, the prop tables, and the design principles (no borders, no rounded corners, `Filler` instead of `justify-*`). Read it before laying out anything.

2. **Any other component** (button, badge, input, select, tabs, switch, slider, separator, kbd, avatar, tooltip, popover, dropdown-menu, collapsible, …) — there is **no per-component `.md`** doc; the authoritative, never-stale source is:
   - the registry entry — `https://hub.joyco.studio/r/<name>.json` (gives the title, description, dependencies, and the file path), and
   - **the installed source itself** — `@/components/ui/<name>.tsx` in the project. Read it for the real variants, sizes, and props. This is the ground truth.

3. **The full catalog** of what's installable (base components *and* JOYCO signature pieces — marquee, magnetic, scramble, media/canvas, …):
   `https://hub.joyco.studio/r/registry.json`
   Fetch it instead of guessing component names. Install by slug: `pnpm dlx shadcn@latest add @joyco/<name>`.

If a fact you need isn't in any of those, it isn't a kit fact — don't invent it.

---

## What this skill won't repeat

The Cluster/Filler concept, every component's variants and sizes, and the kit's styling rules are documented at the sources above — most thoroughly in `cluster.md`. Read them there rather than relying on anything restated here. In particular, `cluster.md` is the source of truth for the layout philosophy (transparent containers, children own their background, no borders / radius `0rem`, `Filler` over `justify-between`). Follow it; don't paraphrase it from memory.

---

## Pitfalls

These are the mistakes Claude makes when it skips the live docs and works from generic shadcn habits:

- **Building a card** as a `<div>` with `bg-*` + `p-*`. JOYCO containers are transparent; children own their background. See `cluster.md`.
- **`justify-between` / `justify-end` / big margins** to push things apart, instead of `<Filler />`.
- **Rounded corners or borders** on layout elements. The theme radius is `0rem`; borders are essentially unused (outline button is the lone exception). This trips up shadcn muscle memory — always check `cluster.md`.
- **Guessing a component's variant or prop names** instead of reading `@/components/ui/<name>.tsx`.
- **Per-internal `className` props** instead of styling via `data-slot` from the parent (`**:data-[slot=name]:…`).
- Icon-only buttons without `aria-label`; removing focus outlines without a `focus-visible` replacement.

---

## Checklist

- [ ] Read `cluster.md` from the hub before doing layout — didn't reconstruct Cluster/Filler from memory.
- [ ] For any component used, confirmed its real API from `@/components/ui/<name>.tsx` (or the registry JSON), not from a remembered table.
- [ ] Layout uses `Cluster` + `Filler`; no faked cards, no `justify-*` spacing, no borders/rounded corners.
- [ ] New components pulled by slug from the live registry, not assumed to exist.
- [ ] Internals styled via `data-slot` from the parent; single `className` on the root.
