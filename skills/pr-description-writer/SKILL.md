---
name: pr-description-writer
description: >
  Generate high-quality Pull Request descriptions in Markdown.
  Supports issue, feature, and big-feature PR types with structured,
  production-ready output. Also generates PR titles and supports
  delivery as chat output or downloadable .md file.
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# PR Description Writer Skill

This skill generates clean, structured, production-ready Pull Request descriptions in Markdown format.

The user provides context about their PR (changes, bug, feature, architecture, etc). The skill determines or asks for the PR type and generates the correct template.

---

## PR Types

Supported types:
- `issue`
- `feature`
- `big-feature`

If the user does not specify the type, ask them to choose one.

---

## General Writing Rules

- Write in English.
- Be concise and high-signal.
- Avoid generic filler text.
- Focus on **why + impact**, not only what changed.
- Use clean, professional technical writing.
- Prefer clear technical explanations over marketing language.

---

## PR Title Rules

The skill must always generate a PR title before the description.

Rules:
- Must be concise and descriptive
- Must reflect intent and impact (not only change)
- Prefer conventional PR style when possible

Examples:
- `fix(auth): prevent session reset on refresh`
- `feat(payments): add MercadoPago checkout support`
- `refactor(ui): migrate layout system to grid tokens`

---

## Output Mode

This skill ONLY generates Markdown output.

It always:
- Generates PR title
- Generates PR description
- Formats everything as Markdown
- Is ready for manual copy/paste into GitHub

---

## Final Output Format

```md
# <PR Title>

<PR Description using the selected template>
```

---

## Linear Issue / Issue Link

Always ask if there is a related issue (Linear, Jira, GitHub Issues, or similar).

If an issue exists, add this section at the top of the PR description:

```md
## Issue

[Issue link here]
```

If there is no issue, omit this section.

---

## Template --- Issue

Use for:
- Bugs
- Fixes
- Regressions
- Performance problems
- Technical corrections

```md
## Problem

<Explain the bug or problem clearly. What was happening? When did it occur?>

## Root Cause

<Explain the technical cause if known.>

## Solution

<Explain what was changed and why it fixes the problem.>

## Result

<Explain impact. Add demo links, videos, screenshots if available.>
```

---

## Template --- Feature (Small / Medium)

Use for:
- Normal features
- Improvements
- Small UX additions
- Non-architectural changes

```md
## Problem

<Explain the need, limitation, or missing capability.>

## Root Cause

<Optional. Only include if there was a technical limitation to solve.>

## Result

<Explain what the new feature enables and its impact.>
```

Rules:
- Do NOT include **Solution** section for features.
- Root Cause is optional.

---

## Template --- Big Feature

Use for:
- New systems
- Architectural changes
- Major UX flows
- New platform capabilities
- Large refactors

```md
### Summary

<High level explanation of what this PR introduces and why it matters.>

### Key Changes

- <Main change>
- <Main change>
- <Main change>

---

## Architecture Overview

### 1. Concept / Strategy

<Explain main architectural idea>

### 2. Implementation

<Folders, routing, state, patterns, infra, etc>

### 3. Components / Systems

<Main modules or domains involved>

### 4. Data / State Flow

<How data moves through the system>

### 5. Navigation / UX Flow

<If applicable>

### 6. Migration / Breaking Changes

<If applicable>

---

### Notes

<Tradeoffs, future improvements, known limitations>
```

---

## Output Delivery

After generating the PR Title and PR Description, the skill must ask the user how they want to receive the output.

Ask:

"Do you want me to generate a `.md` file with this content, or just show it here?"

Options:
- Show in chat (default)
- Generate `.md` file

---

## Markdown File Rules

If the user requests a `.md` file:

- The file must contain:
  - PR Title as H1
  - PR Description body
- No extra text outside Markdown
- File must be ready to upload or commit

---

## Behavior Rules

When running this skill:

1. Detect PR type from context OR ask user to choose:
   - issue
   - feature
   - big-feature

2. Always ask if there is a related issue link.

3. If issue exists, add Issue section at the top.

4. Generate:
   - PR Title
   - PR Description (Markdown body)

5. Ask how to deliver the output:
   - Show in chat
   - Generate `.md` file

6. If `.md` file is requested, generate file-ready Markdown content.

---

## Quality Bar

The generated PR description must:
- Be copy-paste ready for GitHub PR
- Be technically accurate
- Be easy to scan
- Be useful for reviewers and future maintainers
