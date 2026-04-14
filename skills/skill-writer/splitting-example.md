# Splitting Example — `webaudio` Effects

A real before/after from this repo: extracting the "writing reusable effect modules" guidance out of `webaudio/SKILL.md` into a sibling `writing-effects.md`. Use this as a template for the same kind of refactor on other skills.

## The trigger

`webaudio/SKILL.md` had a short "Effect chain" recipe that showed how to build a one-off filter and route voices through it:

```md
### Effect chain (filter, reverb, etc.)

Build the chain once, reuse the head node across many plays:

```ts
const filter = ctx.createBiquadFilter()
…
const fx = suno.effect(filter)
suno.get('click').play({ output: fx })
```
```

That's a recipe — it belongs in `SKILL.md`. But a separate guideline came in describing *how to author reusable effect classes*: SSR-safety rules, `attach()` / `detach()` lifecycle, buffered params, parallel topologies, a six-rule checklist, ~180 lines of prose and code.

Inlining all of that would have:
- More than doubled `SKILL.md`'s length.
- Buried the simple recipes (one-shot SFX, Mixer, slowmo) under a wall of class-design rules most callers never need.
- Mixed two audiences — *users* of effects (every Suno consumer) and *authors* of effect modules (a smaller subset).

All three split criteria from `SKILL.md` met:
1. Reference material the agent re-reads on demand. ✓ (an authoring spec with rules and a checklist)
2. Long enough to bury other content. ✓ (~180 lines)
3. Clear handoff point — the existing "Effect chain" recipe is the natural pivot. ✓

## The split

1. Created `skills/skills/webaudio/writing-effects.md` containing the full authoring guideline.
2. Left the existing "Effect chain" recipe in `SKILL.md` untouched — it's the right level of detail for inline use.
3. Added one paragraph at the end of that recipe, pointing onward:

```md
For reusable effect modules (classes that wrap a node graph, expose a
`chain` head, and are safe to declare at module scope alongside `suno`),
follow the pattern in [`writing-effects.md`](./writing-effects.md). It
covers SSR-safe construction, `attach()` / `detach()` lifecycle, buffered
params, parallel topologies (dry/wet), and per-voice routing.
```

The link sits *exactly* where an agent would already be reading if they were thinking about effects, so the handoff is natural — they don't have to scan a "see also" section at the bottom.

## What the link blurb has to do

- **Name the audience** ("reusable effect modules", "classes that wrap a node graph") so the agent can decide whether to follow it. Not every effect-related task needs the deeper doc.
- **Summarize what's inside** in one sentence — the agent uses this to predict whether the doc has the answer before opening it.
- **Use a relative path** (`./writing-effects.md`) so previews and editors resolve it.

A bare "see writing-effects.md" link isn't enough. The agent has to choose to open it without seeing the contents.

## What stays in `SKILL.md`

- The recipe itself (the short, immediately-useful version).
- The pointer to the deeper doc.

What does *not* stay in `SKILL.md`:
- The six rules.
- The canonical class skeleton.
- The parallel-graph template.
- The checklist for authoring effects.

All of those are reference material — they're consulted when authoring, not when reading the skill end-to-end.

## When you'd undo this split

If `writing-effects.md` ever shrank to two paragraphs (because the rules were superseded by a different abstraction, say), inline it back. A 30-line sibling that gets opened every time the parent is read costs more than it saves. Splits are reversible — re-evaluate when the underlying material changes shape.
