# JOYCO Agent Skills

This repo contains the agent skills for JOYCO.

## Install

```sh
pnpx skills add joyco-studio/skills
```

## Curation rule

Keep a skill here when it captures a JOYCO-specific workflow, a stable
cross-project convention, or a structured analysis that an agent cannot infer
reliably on its own. Prefer the live [logs](https://hub.joyco.studio/logs) and
[toolbox](https://hub.joyco.studio/toolbox) for library APIs and evolving tool
instructions.

Do not add skills for generic engineering work already handled by modern agent
harnesses, or for static copies of fast-changing package documentation.

## Skills

| Skill | Description |
|-------|-------------|
| [joyco-app](skills/joyco-app/SKILL.md) | Carry a new JOYCO internal Next.js product app from plan to working build: lock a plan first, run the opinionated kickoff (JOYCO UI shadcn kit — theme, fonts, `@joyco/*` components from hub.joyco.studio — plus lint config and agent guidance), then build it out to the plan. User-invocable as `/joyco-app`. |
| [joyco-logs](skills/joyco-logs/SKILL.md) | Before implementing or planning a non-trivial feature, scan the JOYCO dev-team logs (`hub.joyco.studio/logs`) and toolbox (`hub.joyco.studio/toolbox`) for a relevant write-up or maintained tool and use it first. Surfaces the team's gotchas/patterns and CLIs/libraries, defers to dedicated skills on overlap, and credits the source to the user. |
| [figma-static-pass](skills/figma-static-pass/SKILL.md) | Build the static, production-correct version of a Figma frame before any animation — layout-system translation, semantic HTML, theme tokens, asset handling, and the `data-motion` naming contract the motion pass resolves against. Stage one of the motion-handoff pipeline. |
| [figma-motion-handoff](skills/figma-motion-handoff/SKILL.md) | Translate Figma Motion animations into production code (CSS/Tailwind, Motion for React, anime.js, or GSAP). Covers canvas-to-layout translation, node mapping, easing-token extraction, paint discipline, and when to stop and ask instead of guessing. Stage two; lands on top of `figma-static-pass`. |
| [review-motion-handoff](skills/review-motion-handoff/SKILL.md) | Audit an animation implementation against its Figma Motion source on two axes — fidelity (matches the design) and quality (performant, accessible, idiomatic). Produces a severity-sorted findings table and verdict, not a rewrite. |
| [joyco-lab](skills/joyco-lab/SKILL.md) | Automates extracting self-contained experiments from any codebase, scaffolding with JOYCO templates, deploying to Vercel, and publishing to the JOYCO Lab registry. Handles 3D, animation, and interactive code experiments. |
| [thrash-report-analyzer](skills/thrash-report-analyzer/SKILL.md) | Analyze a bye-thrash layout thrashing report array. Parses stack traces, identifies user-code functions causing forced reflows, locates the offending style-write → layout-read pairs in source files, and produces fix suggestions. |
| [trace-audit](skills/trace-audit/SKILL.md) | Analyze a Chrome DevTools Performance trace JSON file for performance anomalies, producing a structured audit report with critical issues, warnings, metrics, timeline hotspots, and actionable recommendations. |
| [skill-writer](skills/skill-writer/SKILL.md) | Author or refactor a skill in this repo. Covers frontmatter conventions, file layout, and the rule for splitting deep reference material into linked sibling docs instead of bloating `SKILL.md`. |
| [joyco-ui](skills/joyco-ui/SKILL.md) | Use the JOYCO UI kit correctly when building interfaces. A thin router that sends the agent to the live hub docs and installed source (`cluster.md`, the `@joyco` registry, the component `.tsx`) instead of restating the API, so it never goes stale. Pairs with `joyco-app`. |
