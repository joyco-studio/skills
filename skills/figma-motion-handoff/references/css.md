# Target: CSS (with or without Tailwind)

## Easing and duration tokens: placement rules

**Never hardcode `cubic-bezier()` inline in component code or utility classes.** Easing curves are design tokens. The whole point of the handoff is that the curve the designer picked survives as a named, reusable value.

### Tailwind v4 (CSS-first config)

1. Locate the global stylesheet: the file containing `@import "tailwindcss"`. Common paths: `app/globals.css`, `src/app/globals.css`, `src/index.css`, `styles/globals.css`. Search for `@import "tailwindcss"` if not obvious.
2. Add easing variables inside the `@theme` block using the `--ease-*` namespace. This auto-generates `ease-{name}` utilities:

```css
@theme {
  --ease-expand: cubic-bezier(0.22, 0.28, 0.05, 1);
  --ease-recede: cubic-bezier(0.5, 0, 0.5, 1);
}
```

3. Durations follow the same pattern with `--duration-*` (generates `duration-{name}` utilities). Only tokenize durations that repeat or carry intent; a one-off 2s timeline can stay in the `@keyframes`/`animation` declaration.
4. Custom `@keyframes` used by Tailwind animation utilities go inside `@theme` too, attached to `--animate-*` entries. Standalone `@keyframes` (not exposed as a utility) can live at the stylesheet root, near a comment naming the source Figma frame.
5. If the project has no `@theme` block yet, add one to the global stylesheet; do not create a second stylesheet.

### Tailwind v3 (JS config)

Extend `theme.extend.transitionTimingFunction` and `theme.extend.transitionDuration` in `tailwind.config.{js,ts}`. Keyframes and animations go in `theme.extend.keyframes` / `theme.extend.animation`.

### Plain CSS (no Tailwind)

Define tokens as custom properties on `:root` in the global stylesheet, same naming scheme (`--ease-expand`), and reference them everywhere: `animation-timing-function: var(--ease-expand);`.

## Translating Figma Motion keyframes to @keyframes

- Timeline duration → `animation-duration`.
- Keyframe `timeMs / durationMs` → percentage offsets.
- Hold frames: repeat the same value at both boundary percentages (e.g. value at `0%, 20%` then change starts). CSS has no hold easing; flat segments encode it.
- Per-property easing: CSS `@keyframes` applies `animation-timing-function` per keyframe segment, and you can set it inside a keyframe block to control the curve leaving that frame:

```css
@keyframes recede {
  0%, 20% { opacity: 1; animation-timing-function: var(--ease-recede); }
  50%, 100% { opacity: 0.4; }
}
```

- If two properties on the same element need different curves (common in Figma Motion output), split into two `@keyframes` and comma-compose: `animation: move 2s var(--ease-expand), dim 2s var(--ease-recede);`. Property overlap between the two must be zero.
- Loop: `animation-iteration-count: infinite` only after confirming loop intent.
- Transforms compose in one property. Merge Figma's separate scaleX/scaleY/translateX/translateY tracks into a single `transform` per keyframe. If their curves differ, use individual transform properties (`translate`, `scale`, `rotate`), which animate independently with their own timing functions.

## Springs

Figma Motion spring easings, or beziers you agreed to treat as springs, map to the CSS `linear()` function. Generate a `linear()` approximation (15-30 stops) and store it as a token the same way. If browser support for `linear()` is a concern for the project, fall back to the closest bezier.

## Tier 2 in CSS: View Transitions

Layout-dependent animations (expand to fullscreen, shared element) use the View Transitions API. The Figma data contributes duration and easing only:

```css
::view-transition-group(card) {
  animation-duration: 0.6s;
  animation-timing-function: var(--ease-expand);
}
```

Assign `view-transition-name` to the moving element in both states. Sibling treatment (blur/dim of everything else) is usually Tier 1 and ships as a plain class-driven transition alongside.

If View Transitions are unavailable or the project already has a FLIP utility, defer to that; measurement rules in `references/performance.md`.

## Reduced motion

Wrap non-essential animation in `@media (prefers-reduced-motion: no-preference)`, or provide a reduced variant that keeps opacity fades but removes spatial movement. Tailwind: `motion-safe:` / `motion-reduce:` variants.

## Blur and filter cost

`filter: blur()` animations (Figma's `effect[0].radius`) are paint-heavy. Keep blurred areas small, cap radius, and see the budget rules in `references/performance.md` before shipping a blur on a large surface.

## Initial states

- Mount animations with delays (staggered cohorts): `animation-fill-mode: backwards` (or `both`) so the first keyframe's styles apply during the delay. Without it, delayed items flash their resting state.
- Scroll/late-triggered reveals: hidden initial state lives in the stylesheet on the `data-motion` selector, gated by `@media (prefers-reduced-motion: no-preference)` and a JS-availability gate (`html.js` class set inline in <head>, or `@media (scripting: enabled)` where support allows). The trigger callback then starts the animation imperatively (WAAPI `element.animate` with the token values). Never toggle a class or `data-visible` attribute into a CSS transition to run the reveal; selector flips force wide style recalculation (see references/performance.md).
