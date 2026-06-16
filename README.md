# JOYCO Agent Skills

This repo contains the agent skills for JOYCO.

## Install

```sh
pnpx skills add joyco-studio/skills
```

## Skills

| Skill | Description |
|-------|-------------|
| [joyco-app](skills/joyco-app/SKILL.md) | Carry a new JOYCO internal Next.js product app from plan to working build: lock a plan first, run the opinionated kickoff (JOYCO UI shadcn kit — theme, fonts, `@joyco/*` components from hub.joyco.studio — plus lint config and the Claude Code agent), then build it out to the plan. User-invocable as `/joyco-app`. |
| [lab-experiment](skills/lab-experiment/SKILL.md) | Automates extracting self-contained experiments from any codebase, scaffolding with JOYCO templates, deploying to Vercel, and publishing to the JOYCO Lab registry. Handles 3D, animation, and interactive code experiments. |
| [parallel-claudes](skills/parallel-claudes/SKILL.md) | Guide for running multiple parallel Claude Code sessions using `cw` (Claude Worktree Manager). Covers git worktree management, concurrent AI coding tasks, and merging results. |
| [pr-description-writer](skills/pr-description-writer/SKILL.md) | Generate high-quality PR descriptions in Markdown. Supports issue, feature, and big-feature PR types with structured, production-ready output. |
| [markdown-content](skills/markdown-content/SKILL.md) | Guide for building markdown content pages in Next.js using fumadocs-mdx, rehype-pretty-code, and shiki. Covers MDX collections, prose typography, code blocks, and component consistency. |
| [thrash-report-analyzer](skills/thrash-report-analyzer/SKILL.md) | Analyze a bye-thrash layout thrashing report array. Parses stack traces, identifies user-code functions causing forced reflows, locates the offending style-write → layout-read pairs in source files, and produces fix suggestions. |
| [trace-audit](skills/trace-audit/SKILL.md) | Analyze a Chrome DevTools Performance trace JSON file for performance anomalies, producing a structured audit report with critical issues, warnings, metrics, timeline hotspots, and actionable recommendations. |
| [webaudio](skills/webaudio/SKILL.md) | Add sound effects, UI audio, and ambient sound to a web app using `@joycostudio/suno`. Picks the right entry point (vanilla vs React, with/without Mixer), writes a typed manifest, and wires an unlock gesture. |
| [skill-writer](skills/skill-writer/SKILL.md) | Author or refactor a skill in this repo. Covers frontmatter conventions, file layout, and the rule for splitting deep reference material into linked sibling docs instead of bloating `SKILL.md`. |
| [react-ts-debugging](skills/react-ts-debugging/SKILL.md) | Debug React and TypeScript bugs through a collaborative, log-driven loop with the developer instead of guessing from source code. Strategic console logs drive each next step. |
| [resolving-git-conflicts](skills/resolving-git-conflicts/SKILL.md) | Diagnose and resolve git conflicts of any kind — merge, rebase, cherry-pick, stash, revert. Detects phantom conflicts from squash merges or rewritten history and picks the right git strategy instead of hand-editing markers. |
| [joyco-ui](skills/joyco-ui/SKILL.md) | Use the JOYCO UI kit correctly when building interfaces. A thin router that sends Claude to the live hub docs and installed source (`cluster.md`, the `@joyco` registry, the component `.tsx`) instead of restating the API, so it never goes stale. Pairs with `joyco-app`. |
