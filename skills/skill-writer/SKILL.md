---
name: skill-writer
description: >
  Author or refactor a skill in this repo. Use when the user asks to "create
  a skill", "write a skill", "add a new skill", "document this as a skill",
  or to restructure an existing SKILL.md (split it up, slim it down, fix
  the frontmatter). Covers frontmatter conventions, file layout, and the
  rule for splitting deep reference material into linked docs instead of
  bloating SKILL.md.
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# Skill Writer

A skill in this repo is a self-contained directory under `skills/<name>/` with one entry-point file (`SKILL.md`) and any number of supporting docs alongside it. The harness loads `SKILL.md`'s frontmatter to decide when to surface the skill; the body is what the agent reads when it triggers.

This skill is for writing those entries well: tight `SKILL.md`, deep material lifted into siblings.

---

## When to use

- The user asks to author a new skill, or to refactor an existing one.
- A `SKILL.md` is growing past ~250 lines and needs to be split.
- You're about to add a large reference table, threshold list, full template, or worked example to a `SKILL.md`.

If the user just wants to *use* an existing skill, invoke that skill directly. This skill is for *authoring*.

---

## Curation gate

Before writing or expanding a skill, confirm the content belongs in this
repository. A skill should encode at least one of:

- A JOYCO-specific workflow or policy that changes how work is performed.
- A stable cross-project convention an agent cannot infer reliably.
- A structured input, analysis, or output contract with domain-specific rules.

Do not create a skill when the material is:

- Generic engineering work already handled by the active agent harness.
- A static copy of a fast-changing library or CLI API. Link to its live docs or
  use a thin router skill when discovery itself is valuable.
- A duplicate of a maintained JOYCO log, toolbox page, plugin skill, or project
  document. Improve the authoritative source instead.

Apply the same gate during refactors. If an existing skill no longer clears it,
retire the skill and update every catalog or router that names it.

---

## File layout

```
skills/<name>/
  SKILL.md              # required — entry point and frontmatter
  <topic>.md            # optional — deep reference, templates, examples
  <another-topic>.md
```

One directory per skill. The directory name matches the skill's `name` field, kebab-case. Existing examples to model after:

- **Lean, single file**: `skills/joyco-ui/SKILL.md`, `skills/review-motion-handoff/SKILL.md`.
- **Split into linked docs**: `skills/trace-audit/` (SKILL.md + `detection-heuristics.md` + `report-format.md`), `skills/figma-motion-handoff/` (SKILL.md + target-library references).

---

## Frontmatter

Required:

```yaml
---
name: <kebab-case-id>          # matches directory name
description: >                 # the trigger text — see below
  One paragraph that names the task and lists the verbs/phrases that
  should make an agent reach for this skill.
---
```

Conventional in this repo:

```yaml
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
```

For user-invocable slash commands (the user types `/skill-name <args>`):

```yaml
user-invocable: true
argument-hint: <expected-arg-shape>
```

### Writing the description

The description is the only thing the harness sees when deciding whether to load the skill. Treat it as a trigger spec, not marketing copy:

- Lead with the task ("Analyze a Chrome DevTools trace…", "Publish an experiment…").
- List the user phrases that should fire it ("triggers on 'isolate this', 'publish to lab', …").
- Mention the libraries / file types / domain terms that imply this skill ("Chrome trace JSON", "Figma Motion", "`@joyco/ui`").
- Say what *not* to fire on, if there's a near neighbor that gets confused.

Look at `joyco-lab` for a description heavy on trigger phrases, and `thrash-report-analyzer` for one heavy on file-format and domain terms. Pick the form that matches how users will reach for the skill.

---

## The split rule

Default: write everything in `SKILL.md`. Split a section into its own sibling `.md` only when **all** of these hold:

1. The section is reference material the agent re-reads on demand (a heuristics table, a template, a full worked example, a per-topic deep-dive).
2. It's long enough that inlining it would bury the rest of `SKILL.md` (rule of thumb: ≥80 lines, or its own table of contents starts to make sense).
3. The skill has a clear "do step N — see `<topic>.md`" handoff point, so the agent knows when to open it.

Don't split for the sake of it. A 250-line `SKILL.md` that reads top-to-bottom is better than a 60-line stub that punts every section to a sibling — splitting costs the agent a tool call and fragments the mental model.

For a concrete before/after walkthrough of pulling a section out of a fat `SKILL.md`, see [`splitting-example.md`](./splitting-example.md).

### Linking siblings

From `SKILL.md`, link with a relative path so editors and previews resolve it:

```md
See [`detection-heuristics.md`](./detection-heuristics.md) for the full pattern list.
```

In step-by-step workflows, name the file inline at the step that needs it (see `trace-audit/SKILL.md` — "Refer to `detection-heuristics.md` for the full set of patterns"). The agent reads the workflow top-to-bottom; tell it exactly when to open the sibling.

---

## SKILL.md body shape

There's no rigid template, but the skills that work well share this skeleton:

1. **One-line purpose** — what this skill does, restated from the description in the agent's voice.
2. **When to use / not to use** — bulleted, concrete. Helps the agent self-correct.
3. **Mental model or prerequisites** — the conceptual frame or the env requirements (CLI tools installed, accounts configured) the agent needs before doing anything.
4. **The actual instructions** — either a numbered workflow (for slash-command-style skills like `trace-audit`) or topic sections (for guide-style skills like `joyco-ui`).
5. **Pitfalls / common mistakes** — short list of things the agent will get wrong if not warned.
6. **Final checklist** — the "before you finish" gate. Keep it short; every item should be a real failure mode.

Skip any section that isn't earning its keep. A guide skill might not need a workflow; a workflow skill might not need a mental model.

---

## After writing the skill

1. Add a row to the table in [`/README.md`](../../README.md) — title links to the new `SKILL.md`, description is one or two sentences lifted from the frontmatter (not the full paragraph).
2. Verify the directory contains exactly the files you intended (`ls skills/<name>/`).
3. Read `SKILL.md` straight through. If your eyes glaze on a section, that's a split candidate (or a delete candidate).

---

## Checklist

- [ ] Directory name matches `name` in frontmatter (kebab-case).
- [ ] Frontmatter has `name` and `description`; `description` lists trigger phrases or domain terms.
- [ ] `license`, `metadata.author`, `metadata.version` set (repo convention).
- [ ] If user-invocable, `user-invocable: true` and `argument-hint` present.
- [ ] No deep reference material inlined that should be a sibling `.md`.
- [ ] Sibling `.md` files are linked from `SKILL.md` with a relative path at the step that needs them.
- [ ] `README.md` table updated.
- [ ] `SKILL.md` reads cleanly top-to-bottom.
