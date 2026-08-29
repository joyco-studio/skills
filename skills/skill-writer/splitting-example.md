# Splitting Example — Trace Audit References

The `trace-audit` skill shows when detailed reference material should live next
to `SKILL.md` instead of inside it.

## The trigger

The workflow needs two large bodies of information:

- Detection patterns and severity thresholds for every performance category.
- A complete output template for the final audit.

Both are necessary during specific steps, but neither helps an agent decide
whether the skill applies or understand the workflow at a glance. Inlining them
would bury six short workflow steps under tables, examples, and a long report
template.

All three split criteria are met:

1. They are reference material consulted on demand.
2. They are long enough to dominate the entry point.
3. The workflow has clear handoff points for each document.

## The split

The skill keeps its decision-making and sequence in `SKILL.md`, then points to
the references exactly where they become necessary:

```md
### Step 3 — Run detection passes

Refer to [`detection-heuristics.md`](./detection-heuristics.md) for the full set
of patterns and thresholds.
```

```md
### Step 6 — Generate report

Output the report using the structure defined in
[`report-format.md`](./report-format.md).
```

The directory stays cohesive:

```text
skills/trace-audit/
  SKILL.md
  detection-heuristics.md
  report-format.md
```

## What the link blurb must do

- Name the task that requires the reference.
- Summarize what the document contains.
- Use a relative link so editors and previews resolve it.
- Sit at the workflow step where the agent must open it.

A generic “see also” section at the bottom is weaker because the agent must
guess whether and when the extra context matters.

## What stays in `SKILL.md`

- Trigger and usage contract.
- Ordered workflow.
- Short category summary.
- Final quality bar.

What moves out:

- Exhaustive pattern and threshold tables.
- Long templates.
- Detailed examples used only during one workflow step.

If a reference later shrinks to a few paragraphs and is needed on every run,
inline it again. Splits are reversible and should continue earning their extra
tool call.
