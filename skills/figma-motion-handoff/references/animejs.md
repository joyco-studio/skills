# Target: anime.js (v4)

Check the installed major version first (`package.json`). v4 changed the API surface (`animate`, `createTimeline`, named imports); v3 uses the single `anime()` function. Below assumes v4; adapt if v3.

## Token placement

Same principle as the other targets: easings and durations are named tokens in a shared module, e.g. `lib/anim.ts`:

```ts
export const EASE = {
  expand: 'cubicBezier(0.22, 0.28, 0.05, 1)',
  recede: 'cubicBezier(0.5, 0, 0.5, 1)',
} as const

export const DUR = {
  reveal: 600,
} as const
```

anime.js takes durations in ms (unlike Motion's seconds); keep tokens in ms and note the unit in the name or a comment if both libraries coexist.

If the codebase mixes anime.js with Tailwind `--ease-*` variables, mirror names 1:1.

## Translating Figma Motion keyframes

- Easing: `cubicBezier(x1, y1, x2, y2)` string for designed curves. Only use penner names (`'outQuint'`, `'inOutSine'`) when the user asked to approximate; the designed bezier is the source of truth.
- Keyframes: per-property keyframe arrays with per-keyframe `duration` and `ease`:

```ts
import { animate } from 'animejs'
import { EASE } from '@/lib/anim'

animate('.card', {
  scale: [
    { to: 1, duration: 400 },          // hold: flat segment
    { to: 1.42, duration: 600, ease: EASE.expand },
    { to: 1.42, duration: 1000 },      // trailing hold
  ],
  x: [
    { to: 0, duration: 400 },
    { to: 472, duration: 600, ease: EASE.expand },
    { to: 472, duration: 1000 },
  ],
})
```

- Hold frames become segments whose `to` equals the previous value; duration carries the hold length. This maps 1:1 from Figma's hold easing.
- Convert Figma `timeMs` to per-segment durations: `segment duration = next.timeMs - current.timeMs`.
- Multi-element choreography (timeline cohorts from `get_motion_context`) → `createTimeline()` with absolute position offsets (`timeline.add(target, params, 400)`), which maps directly to Figma's absolute keyframe times. Prefer absolute offsets over relative `'<'`/`'+='` when porting from Figma, the source data is absolute.
- Loop: `loop: true` only after confirming intent. Figma `playbackStyle: 'loop'` in a preview does not mean production loop.
- Transforms: anime.js animates `translateX`, `scale`, etc. as individual transform properties, which matches Figma's separate tracks nicely; keep them separate when curves differ.

## Springs

v4 has `createSpring({ stiffness, damping, mass, velocity })` usable as an `ease`. Same policy as Motion: suggest for interactive gestures, keep the bezier for choreographed sequences unless the user opts in.

## Tier 2: manual FLIP

anime.js has no layout engine, so layout-dependent animations are a manual FLIP:

1. **First**: measure the element's current rect.
2. **Last**: apply the end state (class change, DOM move, fullscreen fixed positioning) and measure again.
3. **Invert**: compute deltas (`dx`, `dy`, `dw/w`, `dh/h`) and apply them as an instant transform so the element appears not to have moved.
4. **Play**: animate the transform back to identity with the designed duration and easing.

Measurement rules are strict, read `references/performance.md`. Summary: batch both measurements, never interleave a write between them, and when `@joycostudio/metri` is in the project, read rects from Metri's cache (`metri.getBounds(el)` / `useBounds`) instead of calling `getBoundingClientRect` directly. For the "Last" measurement after a DOM change, force a targeted `metri.refresh(el)` then read.

Scale-based FLIP distorts border radius and children. Counter-scale inner content (inverse transform on a child wrapper) or animate `clip-path`/`width-height` off the FLIP only if the performance budget allows (it usually does not for fullscreen). Prefer counter-scaling.

The Figma data contributes to the FLIP: phase durations, easing token, delays/holds before the move, and the Tier 1 sibling treatment which ships as a parallel `animate()` call in the same timeline.

## Reduced motion

Gate spatial animations behind `window.matchMedia('(prefers-reduced-motion: reduce)')`; keep opacity-only fallbacks. Check once and branch timeline construction, do not build the full timeline and cancel it.

## Initial states

anime.js has no declarative initial state and runs after paint: setting the from-state in JS guarantees a flash. Hidden initial states ALWAYS live in CSS on the `data-motion` selector (gated by reduced-motion and JS availability, see SKILL.md step 7); timelines animate from there to rest. Never `.set()`-then-animate on late triggers.
