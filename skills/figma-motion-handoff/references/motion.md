# Target: Motion for React (motion / framer-motion)

## Token placement

Easing arrays and durations live in a shared tokens module, not inline in components. Convention: `lib/motion.ts` (or the project's existing equivalent; search for files exporting easings/variants before creating one).

```ts
// lib/motion.ts
export const EASE = {
  expand: [0.22, 0.28, 0.05, 1],
  recede: [0.5, 0, 0.5, 1],
} as const satisfies Record<string, [number, number, number, number]>

export const DUR = {
  reveal: 0.6,
} as const
```

If the project is also on Tailwind and already defines `--ease-*` theme variables, keep the two in sync and note it in a comment; CSS and Motion cannot share one source without a build step, so duplication must be visible and named identically.

## Translating Figma Motion keyframes

The MCP's motion.dev snippet is a good starting shape; clean it up:

- Keyframe value arrays + `times` array (normalized 0-1) map directly from `timeMs / durationMs`.
- Hold frames: duplicate values in the array with distinct `times` entries. The MCP already does this (`[1, 1, 1.42, 1.42]`).
- Per-property transitions: keep the per-property `transition` object shape when curves differ:

```tsx
<motion.div
  initial={false}
  animate={{ scale: [1, 1, 1.42, 1.42], x: [0, 0, 472, 472] }}
  transition={{
    scale: { duration: DUR.reveal, times: [0, 0.2, 0.5, 1], ease: ['linear', EASE.expand, 'linear'] },
    x:     { duration: DUR.reveal, times: [0, 0.2, 0.5, 1], ease: ['linear', EASE.expand, 'linear'] },
  }}
/>
```

- `ease` as an array applies per segment (segments = keyframes - 1). Segments between hold values should be `'linear'` (they are flat anyway).
- Prefer `variants` over inline objects when the animation coordinates multiple elements or has named states; timeline cohorts from `get_motion_context` map naturally to a parent variant with `staggerChildren`/`delayChildren` or explicit per-child `delay`.
- Round the float noise: `1.4215320348739624` → `1.4215` unless the JSON diff step needs exact values.
- Loop: `repeat: Infinity, repeatType: 'loop'` only after confirming intent.

## Springs

For interactive gestures (hover, tap, drag, layout) prefer springs over the designed bezier, and say so in the fidelity note:

```ts
transition={{ type: 'spring', stiffness: 380, damping: 32 }}
```

Heuristic when converting a bezier to a spring: match the settle time to the designed segment duration, keep overshoot only if the bezier's p2y > 1. When the user wants exact fidelity, keep the bezier; beziers are fully supported.

## Tier 2: layout animations and shared elements

This is Motion's home turf. Do not port canvas coordinates; use the layout engine:

- Expansion/reorder within one component tree: `layout` prop. Motion FLIPs it and corrects scale distortion on borderRadius and boxShadow automatically (use `style={{ borderRadius: 6 }}` rather than a class so Motion can correct it).
- Shared element across components (grid card → fullscreen detail): same `layoutId` on both elements; wrap conditional pair in `<AnimatePresence>`.
- The Figma data contributes: duration and easing (`transition={{ layout: { duration: DUR.reveal, ease: EASE.expand } }}`), the choreography (what delays/holds precede the move), and the sibling treatment (Tier 1 blur/dim, ship as a plain `animate` on the siblings with `EASE.recede`).
- If the moving element contains an image, prevent stretching during the FLIP with `layout` on the wrapper and `layout="preserve-aspect"` or a nested `layout` child, verify visually.

Runtime measurement beyond what `layout` handles (e.g. custom scroll-linked values) goes through Metri when available, never raw `getBoundingClientRect` in render or in scroll handlers. See `references/performance.md`.

## Reduced motion

Wrap the app or subtree in `<MotionConfig reducedMotion="user">`, which disables transform/layout animations but keeps opacity for users with the OS setting on. For custom handling use `useReducedMotion()`.

## Initial states and SSR

`initial` handles pre-animation state client-side, but inline styles from `initial` only exist after hydration. For above-the-fold scroll reveals on SSR pages, add the hidden state as a CSS rule on the `data-motion` selector (gated by reduced-motion and JS availability, see SKILL.md step 7) so pre-hydration paint is already hidden; Motion takes over from the same values. For below-fold `whileInView` content, `initial` alone is usually fine.
