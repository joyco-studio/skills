---
name: review-motion-handoff
description: Audit an animation implementation against its Figma Motion source. Use this skill whenever the user asks to review, verify, QA, or diff animation code against a Figma design, pastes Figma Motion Dev Mode JSON next to implementation code, or asks "does this match the design?" about any motion work. Also use it after implementing a Figma Motion handoff to self-check the result.
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# Review Motion Handoff

Audit an animation implementation on two axes: **fidelity** (does it match what was designed?) and **quality** (is it performant, accessible, and idiomatic?). Output a findings table, not a rewrite.

## Inputs

Best case: Figma Motion Dev Mode JSON (ground truth for times, values, curves) plus the implementation code. If only a Figma frame link is available, call `get_motion_context` (recursive) to obtain the keyframe data. If neither exists, say so and limit the review to the quality axis.

## Fidelity checks (against the JSON / motion context)

Normalize before comparing: round beziers to 3 decimals (Figma emits float noise: `0.2199999988` is `0.22`), convert `timeMs` to offsets (`timeMs / timelineDurationMs`), compute relative deltas (`end - start`) from canvas absolutes.

| Check | Pass condition |
|---|---|
| Total duration | Matches `timelineDurationMs` (unit-converted: Motion uses seconds, anime.js and CSS ms) |
| Keyframe offsets | Match `timeMs/duration` within 0.5% (a `times: [0, 0.2001, ...]` vs JSON `400.205ms/2000ms` is a pass) |
| Easing curves | Bezier control points match after rounding; per-property curves not collapsed into one shared curve when the source differs |
| Hold frames | Source holds (`hold: true` / duplicated values) exist as flat segments; not silently dropped |
| Value deltas | Transform deltas equal JSON `end - start` for Tier 1; for Tier 2, values are runtime-computed (FLIP, layoutId, View Transitions) and NOT hardcoded canvas numbers |
| Loop | `playbackStyle` handled deliberately: looping only if confirmed intentional; flag `repeat: Infinity` on transition-like animations |
| Coverage | Every animated node in the source has a counterpart; every choreographed sibling treatment (dim, blur, recede) is present |
| Naming contract | Animated nodes resolve to `data-motion` attributes matching kebab-cased layer names, both directions: every source node has a stamped target, every stamp has source motion data. Orphans on either side are findings |
| Initial states | Elements animating from a non-resting state (opacity 0, offsets, scale 0) have that state applied pre-paint: CSS rule or `animation-fill-mode: backwards` for delayed mounts, stylesheet/`initial` for scroll reveals. Initial state set from JS after paint (anime.js `.set()`, effects writing styles) is a critical finding: first-frame flash. Hidden states must be gated (reduced-motion, JS availability) so content can never be permanently hidden; an ungated `opacity: 0` in the stylesheet is a critical finding |
| Property mapping | Figma fields mapped correctly: `motionTranslationX/Y` → transform translate, `motionScaleX/Y` → scale, `effect[0].radius` → blur, `opacity` 0-100 → 0-1 |

Hardcoded canvas coordinates in layout-dependent animations is a **critical** finding, it will break on any viewport other than the design frame.

## Quality checks

Performance (full detail in the figma-motion-handoff skill, `references/performance.md`, and https://hub.joyco.studio/logs/12-the-render-pipeline plus /logs/11-wtf-is-layout-thrashing):

- Only transform/opacity on the hot path; no animated width/height/top/left/margin where a transform would do.
- Blur/filter animations: flag large blurred surfaces, suggest profiling; blur radius near the source value (12px on a big grid deserves a note).
- No `getBoundingClientRect`/layout reads inside scroll handlers or animation ticks; in projects with `@joycostudio/metri`, measurements go through the cache (`getBounds`, `useBounds`, `useScrollCallback`), and DOM-change-then-measure sequences use targeted `refresh(el)`. Interleaved read/write is a critical finding (layout thrashing).
- If a custom scroll source (Lenis etc.) exists, scroll-linked values must come from it, not `window.scrollY`.
- `will-change` applied temporarily, not permanently.
- Reveal/scroll triggers driven through attribute- or class-selector CSS transitions (`[data-visible="true"] { transition: ... }`, `.is-visible`) are a **major** finding: selector flips force wide style recalculation (repaint path, hub render-pipeline log). The pass condition is an imperative start (WAAPI or the library) with the initial state server-rendered.
- GSAP: `gsap.from`/`gsap.fromTo` on SSR-visible content is a **critical** finding (from-values arrive after server HTML paints: guaranteed flash). Timeline not held in a ref, or created outside `useGSAP` (no cleanup), is a **major** finding. A ScrollTrigger created while Lenis or another custom scroller drives the page, without the scroller wired to `ScrollTrigger.update` and the ticker, is a **major** finding.

Accessibility:

- `prefers-reduced-motion` handled, and handled at the layer that wins: if JS drives inline styles, a CSS media query alone is defeated (see https://hub.joyco.studio/logs/13-working-with-reduced-motion). Spatial motion removed or reduced; opacity fades may remain.

Conventions:

- Easing/duration values exist as named tokens (CSS `--ease-*` theme variables in the global stylesheet for Tailwind projects, shared tokens module for Motion/anime.js), not inline magic numbers. Inline `cubic-bezier()` in component code is a finding even when the values are correct.
- Token names follow the classical easing conventions (`ease-out-quart`, `ease-out-back`), not values; descriptive names only for curves that are not classics.
- Duplicate near-identical curves consolidated into one token.
- Spring easings hand-copied as raw sampled functions (`(t) => 1 - e^(-a*t)(...)`) instead of translated to a named token (spring config, `linear()` token, `createSpring`, `CustomEase`) are a convention finding.

Taste (flag, do not block):

- Entrances ease out, exits ease in; linear reserved for progress/marquee-like loops.
- Duration sanity: micro-interactions under ~300ms; anything over 1s on a UI transition needs a reason (choreographed hero moments are a valid reason).
- Springs on interactive gestures, curves on choreography.

## Output format

A findings table sorted by severity, then a verdict:

| # | Severity | Axis | Finding | Where | Fix |
|---|---|---|---|---|---|
| 1 | Critical / Major / Minor / Note | Fidelity / Perf / A11y / Convention / Taste | one line | file:line or node | one line |

Close with: what is verified 1:1 against the source, what was correctly recomputed for layout, and whether the implementation is approved, approved with notes, or needs changes. Do not rewrite the implementation unless asked; the review's job is the diff.
