---
name: figma-static-pass
description: Build the static, production-correct version of a Figma design before any animation work. Use this skill whenever converting a Figma frame or link into components or pages, setting up the initial state of a demo, scaffolding UI from a design file, or whenever get_design_context output would otherwise be adapted ad hoc. Also use it when a user says a design-to-code result was sloppy or unfaithful and wants it rebuilt properly. It covers layout system translation, the data-motion naming contract, asset handling, token discipline, and the resting-state rule.
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# Figma Static Pass

Turn a Figma frame into production static code. This is stage one of the motion handoff pipeline: the animation pass (figma-motion-handoff skill) lands on top of what this pass produces, so the deliverables here are shippable markup AND the hooks the animation pass will resolve against.

Core premise: **`get_design_context` output is reference material, not shippable code.** It transplants a fixed canvas. Production needs a layout system, semantic HTML, theme tokens, and stable animation hooks.

## Pipeline

### 0. Contract gate (blocking, before any file is touched)

After gathering (step 1), and before creating or modifying ANY file, output a validation table:

| Node id | Layer name | data-motion candidate | Status |

Status is PASS, or FAIL if the layer name is generic: matches patterns like `Group N`, `Frame N`, `Rectangle N`, `Ellipse N`, `Component N`, `Layer N`, `Text` alone, or any default Figma name. Numbered variants of a meaningful name (`card-2`) pass.

If any row is FAIL: **your entire turn is the table plus a request to rename those layers in Figma (or provide names inline). End the turn. Do not build anything, not even the passing nodes.** Building a partial result invalidates the contract for the whole frame and forces rework. This gate exists because the animation pass resolves targets by these exact names; proceeding with invented or generic names silently breaks every future handoff of this file.

If all rows PASS, include the table in your response and continue.

### 1. Gather

- `get_design_context` on the frame: reference code, layer names, node ids, asset URLs, screenshot.
- `get_metadata` if the frame is large and you need the structure map without the full payload.
- `get_motion_context` with `recursive: true`, ONLY to learn which nodes animate. Implement zero animation in this pass. The animated-node list drives the naming contract (step 4) and nothing else.

### 2. Translate the layout system, never transplant the canvas

The Figma frame is one fixed viewport. Decide what actually ships:

- Repeating children (cards, list items, thumbnails) → a driven collection (grid, flex wrap, masonry, list) rendered from data, not N hardcoded siblings.
- Absolute positions from the canvas are only kept for intrinsically overlaid elements (decorative layers, badges pinned to a corner). Everything else flows.
- If the shipped layout system is not obvious (fixed 3-column grid in Figma could ship as responsive grid OR masonry OR carousel), ask the user before building. This single question prevents the most expensive rework.
- Build responsive by default: verify at 375, 768, 1280. The Figma frame width is one data point, not the spec.

### 3. The resting-state rule

Build the design's **resting/final state, fully visible and complete**. Never bake animation initial states (opacity 0, offset positions, scale 0) into the static pass, even when the motion data clearly starts there. Hidden initial states are the animation pass's job, applied with paint discipline and accessibility gates (see figma-motion-handoff). A static page must be complete, readable, and navigable with JavaScript disabled.

If the designed resting state is ambiguous because the frame was captured mid-animation, ask which state is "at rest".

### 4. Stamp the naming contract

For every element whose node appears in the motion context:

- Add `data-motion="<name>"` where `<name>` is the Figma layer name, kebab-cased (`Text Will` → `text-will`).
- Generic layer names cannot reach this step: the step 0 gate already halted on them. If one somehow appears here, treat it as a gate failure and halt the same way.
- One attribute per animated node, on the exact element that moves (not a wrapper, not the component root, unless that IS the animated node).

This attribute map is the contract the figma-motion-handoff skill resolves against deterministically.

### 5. Components and semantics

- Map to existing codebase components first; search before creating.
- Use shadcn/ui primitives where the project has them and one fits the job.
- Semantic HTML: headings are headings, lists are lists, interactive things are buttons/links. Text is text, never an exported image of text.
- One component per repeated design element, props for the varying parts.

### 6. Assets

- Asset URLs from `get_design_context` expire (~7 days). Download them into the project (`public/` or imported) during this pass; never leave hotlinked MCP URLs in code.
- Correct intrinsic dimensions, `alt` text derived from layer names or content, modern formats where the pipeline allows.

### 7. Tokens

- Colors, type sizes, radii, spacing → theme variables (Tailwind v4 `@theme` in the global stylesheet, or the project's token system). No raw hex or magic px in components.
- Before adding a token, check whether an existing one matches within rounding distance; reuse over duplication.
- Map Figma fonts to the project's configured fonts; if the design uses a font the project lacks, ask rather than substituting silently.

### 8. Verify

Compare the rendered result against the design screenshot side by side. Checklist:

- Layout holds at 375 / 768 / 1280, no horizontal scroll.
- Images undistorted, correct aspect ratios.
- Every animated node from the motion context has its `data-motion` stamp; no stamped element lacks a source node.
- Real text selectable, visible keyboard focus on interactive elements.
- No canvas absolutes left on flowing elements.

## Output contract

0. The contract gate table (always first; a FAIL table is the entire output).
1. The components/pages, in the project's conventions.
2. The stamp map: a short table of Figma node → `data-motion` value → file, for the animation pass to consume.
3. Token additions (added vs reused).
4. Deviations: every place the shipped layout intentionally differs from the canvas (layout system chosen, dropped absolutes, responsive decisions), one line each.
