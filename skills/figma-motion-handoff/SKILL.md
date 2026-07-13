---
name: figma-motion-handoff
description: Translate Figma Motion animations into production code (CSS/Tailwind, Motion for React, anime.js, or GSAP). Use this skill whenever the user shares a Figma link to an animated frame, pastes Figma Motion JSON, mentions implementing an animation designed in Figma, or asks to "match the motion in Figma". Also use it when calling the Figma MCP tools get_motion_context or get_design_context for anything animated. It covers node mapping, canvas-to-layout translation, easing token extraction, and when to stop and ask the user instead of guessing.
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# Figma Motion Handoff

Turn Figma Motion data into production animation code without losing design intent and without inventing values that were never designed.

Core premise: **Figma Motion keyframes are canvas-space. Production animations are layout-space.** The translation is rarely 1:1. Your job is to know which parts translate literally, which parts are a spec to recompute at runtime, and which questions to ask when the data cannot tell you.

## Pipeline

Follow these steps in order:

### 1. Detect the target library

Before touching Figma data, inspect the codebase:

- Check `package.json` for `motion`, `framer-motion`, `animejs`, `gsap`.
- Check for Tailwind (`tailwindcss` dependency, `@import "tailwindcss"` or `@theme` in a global stylesheet, or a `tailwind.config.*` file).
- Find where animation values already live: easing tokens in CSS custom properties, a `lib/motion.ts` or `lib/animation.ts` tokens module, existing `@keyframes`, existing variants.
- Check for `@joycostudio/metri`. If present, it is the required tool for any runtime measurement (see `references/performance.md`).

Then read exactly ONE target reference file:
- CSS / Tailwind → `references/css.md`
- Motion for React (framer-motion / motion) → `references/motion.md`
- anime.js → `references/animejs.md`
- GSAP (`gsap` + `@gsap/react`) → `references/gsap.md`

If the codebase has no animation library and the user did not specify one, ask which target they want. Do not pick one silently.

Always read `references/performance.md` before emitting code that animates anything other than transform or opacity, or that requires measuring the DOM.

### 2. Gather the Figma data

Precondition: the static build should already exist (components, layout system, `data-motion` stamps). If it does not, run the figma-static-pass skill first; animating while also inventing layout conflates two jobs and produces canvas transplants.

With a frame link:
1. Call `get_design_context` on the frame node. This gives structure, layer names, node ids, assets, and a screenshot.
2. Call `get_motion_context` with `recursive: true` on the same node. This gives keyframe tracks, pre-computed motion.dev snippets, and timeline cohorts (which nodes share a timeline, duration, loop mode).

With pasted Dev Mode JSON: treat it as ground truth for times, values, and bezier control points, but remember it has no structure, no layer names, and absolute canvas coordinates. Ask for the frame link or a screenshot if you cannot identify what the animated nodes are.

If you have both, use the JSON for exact values and the MCP output for structure and coordination.

### 3. Classify every animated node: Tier 1 or Tier 2

**Tier 1, self-contained.** Transform, opacity, or filter changes on the element itself with no dependency on surrounding layout. Hovers, entrances, loaders, pulsing states, sibling dimming/blur. These translate nearly verbatim. Emit them directly using the target reference conventions.

**Tier 2, layout-dependent.** The animated values depend on runtime layout: expansions to fullscreen, shared element transitions, items moving between containers, reorders, anything inside a responsive grid or masonry. Canvas keyframe values (e.g. `x: 24 -> 496`, `scale: 1 -> 1.4215`) are the designer's illustration on a fixed canvas, not shippable numbers.

For Tier 2, what survives the handoff is the **motion spec**:
- Choreography: order, delays, holds, overlaps (from keyframe times and timeline cohorts)
- Duration of each phase
- Easing curves per property
- Sibling treatment (what happens to everything else)

Values get recomputed at runtime. Each target reference file has a Tier 2 section with the idiomatic technique (View Transitions, `layoutId`, manual FLIP).

Classification signals for Tier 2: the element scales beyond ~1.2x, translation distance is a large fraction of the frame, the end state visually fills the viewport or a container, the element belongs to a repeating grid/list, or the design frame layout will clearly differ from the shipped layout (fixed grid in Figma vs masonry/responsive in code).

### 4. Map Figma nodes to codebase targets (naming contract)

Resolution order, strict:

1. **`data-motion` attribute match.** Kebab-case the Figma layer name (`Text Will` → `text-will`) and look for `[data-motion="text-will"]` in the codebase. Exact match wins, zero guessing. This is the contract the static pass stamps.
2. **Fuzzy match.** No stamp exists: match by layer name vs component/element naming similarity and by rendered content. State the mapping you inferred and stamp the `data-motion` attribute yourself as part of the implementation, so the contract exists going forward.
3. **Ask.** No plausible match: list the animated nodes with their layer names and ask the user to map them. **Never invent a target.** A wrong guess silently attaches an animation to the wrong element.

If source layer names are generic (`Group 2`, `Rectangle 1`), flag them and suggest renaming in Figma; generic names degrade every future handoff of that file.

### 5. Ask before assuming (the question protocol)

Translation is rarely literal. Stop and ask the user when any of these fire. Ask all applicable questions in one batch, not one per message.

- **Absolute endpoints.** Keyframes translate to a fixed canvas coordinate. Ask: is the endpoint viewport-relative (e.g. "center of screen", "fullscreen"), container-relative, or layout-computed?
- **Layout mismatch.** The Figma frame uses a layout the codebase does not (fixed 3-column grid vs masonry, static frame vs scrollable list). Confirm: "the positions in the design are illustrative and I should compute them at runtime, correct?"
- **Spring candidates.** The bezier has a fast attack and long settle (e.g. `[0.22, 0.28, 0.05, 1]`), or the animation is interactive (hover, press, drag, dismiss). Ask whether to keep the exact bezier or convert to a spring. Default to keeping the bezier for choreographed sequences and suggesting a spring for interactive gestures.
- **Loop semantics.** Figma Motion previews often loop for demonstration. Ask whether the loop is intentional in production or just a preview artifact. Never ship `repeat: Infinity` on something that reads like a one-shot transition.
- **Trigger.** Figma Motion does not encode what starts the animation. Ask: on mount, on scroll into view, on click, on hover, on route change?
- **Missing states.** The design shows the animation but not its reverse/exit. Ask whether an exit animation is expected and whether it mirrors the entrance.
- **Responsive behavior.** Ask how the animation adapts on small viewports if the movement is spatial (does it scale down, simplify, or disappear?).

### 6. Extract tokens, never hardcode design decisions

Easing curves and durations are the design decisions that must survive as named tokens:

- Deduplicate curves across the motion data (Figma often emits near-identical beziers with float noise, e.g. `0.2199999988` is `0.22`). Round to 2-3 decimals before comparing.
- Name easing tokens by the classical conventions, family plus direction: `--ease-out-quart`, `--ease-out-back`, `--ease-in-out-expo`. Match the curve to the nearest classic; use a descriptive name only when it genuinely is not one. Never values (`--ease-022-028`). Durations stay descriptive (`--duration-reveal`).
- Check whether an equivalent token already exists in the codebase (globals.css, tokens module, Tailwind theme) before adding a new one. Prefer reusing an existing token if the curve is within rounding distance.
- Where the token lives is target-specific: see the target reference file. The hard rule for CSS/Tailwind projects: easing variables go in the global stylesheet theme, NEVER inline as raw `cubic-bezier()` in component code.

Spring easings are their own case. The Figma MCP can emit a designed spring as a sampled JS function of the form `(t) => 1 - e^(-a*t) * (cos(b*t) + c*sin(b*t))`. That is a designed spring, not a bezier. Never hand-copy the raw function into code; it becomes a named token like any curve. Translation per target:

- Motion for React: `type: "spring"`, deriving damping/stiffness from `a` (decay) and `b` (frequency); verify the feel against the Figma preview.
- CSS: a `linear()` approximation sampled from the function (15 to 30 stops), stored as a token.
- anime.js: `createSpring`.
- GSAP: `CustomEase` from sampled points, or an elastic approximation.

### 7. Initial states and paint discipline

Any animation that starts away from the resting state (opacity 0, offset y, scale 0) can flash its final state before the animation kicks in. The rule: **the initial state must be applied before the first paint of the element's trigger window.** Setting it from JS after mount is always too late; the user sees the resting state for a frame, then a jump. This matters most for scroll reveals and any late trigger, where the flash happens in full view.

By trigger:

- **Mount-triggered CSS**: the `from` keyframe applies at animation start, but delays leave a gap; set `animation-fill-mode: backwards` (or `both`) so the first keyframe styles apply during the delay. This covers staggered cohorts where later items wait.
- **Scroll or interaction-triggered**: the hidden initial state must exist in the stylesheet (e.g. `[data-motion="reveal-card"] { opacity: 0; translate: 0 24px; }`) so server-rendered HTML paints hidden, or come from a framework feature that owns initial state (Motion's `initial`/`whileInView`). With SSR, framework inline styles only exist after hydration; for above-the-fold reveals prefer the CSS rule so pre-hydration paint is already correct.
- **anime.js and imperative libraries**: they have no declarative initial state, so it always lives in CSS. JS only animates toward the resting state.

Trigger mechanism rule, non-negotiable: reveal/scroll triggers must NOT drive the animation through attribute- or class-selector CSS transitions (e.g. `[data-visible="true"] { transition: ... }`). Selector flips force wide style recalculation (see the JOYCO render pipeline log, hub.joyco.studio/logs/12-the-render-pipeline). Instead: the initial state ships in the server-rendered output (CSS rule on the `data-motion` selector, or an inline style matching the animation's from-values exactly), and the trigger callback starts the animation imperatively (WAAPI `element.animate` or the target library). `data-motion` attributes are for TARGETING only, never the animation mechanism.

Safety gates, non-negotiable: a hidden initial state must never be able to permanently hide content. Gate the hidden state so it only applies when the animation will actually run:

```css
@media (prefers-reduced-motion: no-preference) {
  [data-motion="reveal-card"] { opacity: 0; translate: 0 24px; }
}
```

and gate on JS availability for JS-driven reveals (a `js` class set on `<html>` inline in `<head>`, or `@media (scripting: enabled)` where browser support allows). No JS, or reduced motion, means the content simply shows at rest. Prefer opacity/transform for hidden states; never `display: none` or collapsed height for reveals (layout shift when revealed).

### 8. Emit code

Follow the conventions in the target reference file. Universal rules:

- Preserve hold frames (duplicated keyframe values with `hold: true` easing) as flat segments, they are intentional pauses.
- Preserve per-property easing. Figma Motion eases each property independently; do not collapse to a single curve unless the curves are actually identical.
- Convert canvas absolutes to relative deltas for Tier 1 transforms (start value becomes 0 / 1, keyframes become offsets from it).
- Respect the performance rules in `references/performance.md`: animate transform and opacity, treat filter as a paint-cost budget item, never animate layout properties (width, height, top, left, margin, padding) when a transform can do the job.
- Always handle `prefers-reduced-motion` (target-specific mechanism in each reference file).

### 9. Verify

If Dev Mode JSON is available, diff your implementation against it:
- Total duration matches `timelineDurationMs`.
- Keyframe offsets match `timeMs / duration` (allow rounding to 4 decimals).
- Bezier control points match after rounding.
- Hold segments are flat (identical consecutive values).
- Loop mode matches `playbackStyle` (after confirming loop intent with the user).
- Relative deltas: `end - start` from the JSON equals your transform delta.

State explicitly which values are exact and which were recomputed for layout reasons.

## Output contract

Every handoff ends with three things:

1. The implementation (code, using existing project conventions).
2. The token additions (which easing/duration tokens were added or reused, and where).
3. A fidelity note: what is 1:1 with the design, what is runtime-computed, what was confirmed with the user, and anything intentionally deviating (e.g. loop removed, spring substituted).
