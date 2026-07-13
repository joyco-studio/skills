# Target: GSAP

Detection: `gsap` AND `@gsap/react` in package.json. If only `gsap` exists in a React codebase, flag it; `useGSAP` is the required integration, not manual effects.

## Tokens

- Designed beziers register once via `CustomEase.create(name, points)` in a shared module, under the same names as the CSS `@theme` tokens, mirrored 1:1 (classical naming: `out-quart`, `out-back`, `in-out-expo`). One registration site; components reference eases by name string only.
- Durations come from the shared token module. GSAP takes **seconds**; the module documents the unit per export (CSS and anime.js take ms).
- Designed springs: `CustomEase` from points sampled off the spring function, or an elastic approximation when the settle profile matches. Never the raw sampled function inline.

## Mandatory React pattern

Timeline held in a ref outside the effect, created inside `useGSAP` with `scope`, cleanup automatic, replay via `tl.current.restart()`:

```tsx
const scope = useRef<HTMLDivElement>(null)
const tl = useRef<gsap.core.Timeline>(null)

useGSAP(() => {
  tl.current = gsap.timeline({
    scrollTrigger: { trigger: scope.current, start: "top 80%", once: true },
  }).to('[data-motion="reveal-card"]', {
    opacity: 1,
    y: 0,
    duration: seconds(DUR.reveal),
    ease: "out",
  })
}, { scope })
```

- Selectors inside the timeline resolve within `scope`, so `data-motion` targeting stays local.
- Never create timelines in bare `useEffect`; `useGSAP` owns context and cleanup (kills tweens and ScrollTriggers on unmount).

## Initial states

The from-state (opacity 0, offsets) must exist before first paint: a CSS rule on the `data-motion` selector, or an inline `style` attribute in the JSX, matching the timeline's implied from-values EXACTLY.

- ALWAYS `gsap.to` toward rest.
- NEVER `gsap.from` or `gsap.fromTo` for SSR-visible content: JS-applied from-values arrive after the server HTML paints, guaranteeing a flash of the resting state.
- Gate the hidden CSS state behind `prefers-reduced-motion: no-preference` and JS availability, per SKILL.md step 7.

## ScrollTrigger conventions

- Reveals: `once: true`.
- Scroll-linked motion: `scrub` (a number for smoothing, `true` for lockstep).
- `markers` only in dev, never committed enabled.
- Kill/cleanup is `useGSAP`'s job; do not hand-roll `ScrollTrigger.kill()` in effects.
- When Lenis or any custom scroller exists, wire it BEFORE creating any trigger:

```ts
lenis.on("scroll", ScrollTrigger.update)
gsap.ticker.add((time) => lenis.raf(time * 1000))
gsap.ticker.lagSmoothing(0)
```

A ScrollTrigger created against the native scroller while Lenis drives transforms fires at wrong positions or never.

## Reduced motion

Use `gsap.matchMedia`: build the timeline only inside the `(prefers-reduced-motion: no-preference)` context. In the `reduce` context, `gsap.set` the elements to their resting values instantly; content must never depend on the animation running to become visible.
