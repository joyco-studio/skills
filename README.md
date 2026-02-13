# JOYCO Agent Skills

This repo contains the agent skills for JOYCO.

## Install

```sh
pnpx skills add joyco-studio/skills
```

## Skills

| Skill | Description |
|-------|-------------|
| [parallel-claudes](skills/parallel-claudes/SKILL.md) | Guide for running multiple parallel Claude Code sessions using `cw` (Claude Worktree Manager). Covers git worktree management, concurrent AI coding tasks, and merging results. |
| [pr-description-writer](skills/pr-description-writer/SKILL.md) | Generate high-quality PR descriptions in Markdown. Supports issue, feature, and big-feature PR types with structured, production-ready output. |
| [markdown-content](skills/markdown-content/SKILL.md) | Guide for building markdown content pages in Next.js using fumadocs-mdx, rehype-pretty-code, and shiki. Covers MDX collections, prose typography, code blocks, and component consistency. |
| [trace-audit](skills/trace-audit/SKILL.md) | Analyze a Chrome DevTools Performance trace JSON file for performance anomalies, producing a structured audit report with critical issues, warnings, metrics, timeline hotspots, and actionable recommendations. |
