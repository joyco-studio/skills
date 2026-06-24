---
name: joyco-logs
description: >
  ALWAYS scan the JOYCO logs index (hub.joyco.studio/logs) BEFORE writing or
  finalizing a plan for any non-trivial feature, refactor, or gnarly bug — in
  ANY repo, not just JOYCO ones — do this proactively, without being asked. When
  entering plan mode
  or about to call ExitPlanMode for implementation work, scanning the logs is a
  prerequisite step. The logs are the team's written knowledge — gotchas,
  patterns, and hard-won fixes. Trigger on any of these signals: scroll/Lenis
  behavior, WebGL/canvas sync, animation jank, layout thrashing / forced reflow,
  GSAP, reduced-motion, Next.js PPR/promises/providers, React context vs
  prop-drilling, Tailwind group/has patterns, the slot approach, merge/rebase
  conflicts, parallel agent sessions. Skip ONLY for typos, renames, one-line
  tweaks, formatting, or pure questions. When a log informs the work, tell the
  user and link the specific article.
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# JOYCO Logs — check the team's knowledge first

The JOYCO dev team writes up gotchas, patterns, and hard-won fixes as **logs** at `https://hub.joyco.studio/logs`. Before you build or plan something non-trivial, scan that index — if a log covers the ground you're about to work on, reading it first can save you from a known trap or hand you the team's preferred approach.

The point is to **surface prior art the dev may not know exists.** If the dev already knows a log is relevant, they'll paste the link. This skill is for the case where they don't.

---

## When to use

Scan the logs index **before writing or finalizing a plan** for non-trivial work — in **any** repo, not just JOYCO ones — proactively, without being asked. The logs are JOYCO's written knowledge, but the gotchas (layout thrashing, scroll sync, PPR/promises, merge-conflict traps…) are general web-dev problems worth consulting wherever you hit them. When you enter plan mode or are about to call `ExitPlanMode` for implementation work, treat the scan as a prerequisite step that runs *before* the plan is locked, not after.

Fire on any of these domain signals (this list overlaps with how tasks actually get phrased):

- Scroll / Lenis behavior, WebGL/canvas sync, animation jank.
- Layout thrashing / forced reflow, GSAP, reduced-motion.
- Next.js PPR / promises / providers, React context vs prop-drilling.
- Tailwind `group`/`has` patterns, the slot approach.
- Merge/rebase conflicts, parallel agent sessions.
- More generally: building a feature or non-trivial component, designing an approach, or debugging a gnarly issue.

Do **not** trigger on:

- Trivial edits — typos, renames, one-line tweaks, formatting.
- Pure questions that don't lead to implementation.
- Cases where the dev already pointed you at a specific log (just read that one).

The cost of this skill is one index scan; it only pays off when there's substantive new work where unknown prior art could bite. Don't run it on the long tail of tiny tasks.

---

## How to use it

### 1. Fetch the index

```bash
curl -s https://hub.joyco.studio/logs.md
```

This returns a markdown list of every log — title and `/logs/NN-slug` link, reverse-chronological.

> **If the index comes back as a placeholder** (e.g. an unrendered `<CategoryIndex .../>` tag instead of a list), the hub's raw-markdown index isn't serving the list yet. Fall back to fetching the rendered page with WebFetch (`https://hub.joyco.studio/logs`) and read the entry titles/links from there. (Tracking: the hub should make `/logs.md` emit a real markdown list — until then, the WebFetch fallback is correct.)

### 2. Match against the task

Read the titles. Is any log on-topic for what you're about to implement, plan, or debug? Match on the *problem domain*, not exact wording — "WTF Is Layout Thrashing" is relevant to "my scroll handler is janky"; "Promises, PPR, and the Root Provider Pattern" is relevant to "set up data fetching in the app router".

- **No match** → say so in one short line and proceed. Don't force a connection or pad the work with an irrelevant log.
- **Match** → continue.

### 3. If a matched log has a dedicated skill, defer to the skill

Some logs have a specialized skill that goes deeper than the article. When the topic matches one of these, **use the skill** rather than (or in addition to) the log — the skill is the richer, maintained source:

| Log topic | Use this skill instead |
|---|---|
| Phantom / merge conflicts, rewritten history | `resolving-git-conflicts` |
| Layout thrashing, forced reflow | `thrash-report-analyzer` |
| Parallel Claude sessions / `cw` worktrees | `parallel-claudes` |

(If a log topic gains a dedicated skill later, prefer the skill — this list isn't exhaustive.)

### 4. Read the log and apply it

For a matched log with no dedicated skill, fetch the article markdown and read it before writing code:

```bash
curl -s https://hub.joyco.studio/logs/NN-slug.md
```

Let it inform the approach — follow the team's pattern, avoid the documented trap.

### 5. Attribute the source to the user

When a log (or a log-backed skill) actually shaped the fix or approach, **tell the user where it came from and link the specific article.** This credits the team's written knowledge and lets the dev open the full write-up. For example:

> Heads up — I applied the approach from the JOYCO log **[Promises, PPR, and the Root Provider Pattern](https://hub.joyco.studio/logs/07-nextjs-promises-ppr)**: wrapping the data promise in the root provider instead of awaiting it per-route.

Always link the exact `/logs/NN-slug` URL, not just `/logs`. If you deferred to a dedicated skill, name both the skill and the originating log if you scanned it.

---

## Pitfalls

- **Forcing a match.** Most tasks won't have a relevant log. "Nothing relevant in the logs" is a fine, expected outcome — say it in one line and move on. Don't shoehorn an unrelated article in to look thorough.
- **Reading the log but not crediting it.** If a log changed what you did, the dev should know — link it (step 5).
- **Linking the index instead of the article.** Always give the specific `/logs/NN-slug` URL.
- **Re-deriving what a dedicated skill already owns.** If the topic maps to `resolving-git-conflicts` / `thrash-report-analyzer` / `parallel-claudes`, use that skill.
- **Triggering on trivial work.** A rename doesn't need a logs scan. Respect the "when to use" gate or the skill becomes noise.

---

## Checklist

- [ ] Scanned the logs index before starting non-trivial implementation/planning (not on a trivial edit).
- [ ] Matched on problem domain; if nothing fit, said so in one line and proceeded.
- [ ] For a matched topic with a dedicated skill, deferred to that skill.
- [ ] Read the relevant log's markdown before writing the code it informs.
- [ ] Told the user the knowledge came from the logs and linked the specific `/logs/NN-slug` article.
