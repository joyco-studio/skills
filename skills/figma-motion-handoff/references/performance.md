# Performance: budgets, the render pipeline, and Metri

Read this before emitting any animation that touches filters, requires DOM measurement, or animates anything besides transform and opacity.

Primary sources (JOYCO knowledge base, fetch when deeper detail is needed):
- The Render Pipeline: https://hub.joyco.studio/logs/12-the-render-pipeline
- WTF Is Layout Thrashing: https://hub.joyco.studio/logs/11-wtf-is-layout-thrashing
- Performance Monitor: https://hub.joyco.studio/logs/09-performance-monitor
- Reduced motion and GSAP: https://hub.joyco.studio/logs/13-working-with-reduced-motion
- Metri docs: https://hub.joyco.studio/toolbox/metri

## Frame budget and the pipeline

Every animated frame must fit the frame budget (~16.7ms at 60hz, ~8.3ms at 120hz), shared with React work, event handlers, and everything else on the main thread. The pipeline stages are style → layout → paint → composite. The property you animate decides how much of the pipeline re-runs:

- **transform, opacity**: composite only. Runs on the compositor thread, survives main-thread jank. Always the first choice.
- **filter (blur, etc.)**: paint + composite. `blur()` cost scales with blurred area and radius. A 12px blur over a large grid (a common Figma Motion output, `effect[0].radius`) can blow the budget on mid-tier devices.
- **width, height, top, left, margin, padding, font-size, flex/grid properties**: style + layout + paint + composite. Layout can cascade to the whole document. **Never animate these when a transform can express the same motion.** This is exactly why Tier 2 animations are FLIPped: measure once, then animate transform.

Rules of thumb:
- Figma translation/scale tracks → `transform` (or individual `translate`/`scale` properties), never `top/left/width/height`.
- Figma blur tracks: implement as designed, but flag in the fidelity note when the blurred surface is large; propose capping radius, blurring a downscaled layer, or swapping to opacity dim if profiling shows paint cost. Verify with DevTools Performance Monitor (see log 09) or a paint profile.
- `will-change` / forced layers: apply just before a heavy animation, remove after. Never leave permanently.

## Layout thrashing (why measurement discipline matters)

Reading layout (`getBoundingClientRect`, `offsetWidth`, `scrollTop`, ...) after a DOM write forces a synchronous reflow: the browser must run layout immediately to answer. Interleaving reads and writes in a loop or a scroll handler multiplies forced reflows and tanks the frame rate. The fix is batching: all reads first, then all writes (log 11).

FLIP animations and scroll-driven effects are measurement-heavy by nature, which is where Metri comes in.

## Metri (@joycostudio/metri)

A DOM measurement engine: caches element bounds, pools IntersectionObservers, and tracks resize + scroll from a single source, with React bindings. When it is in the project (or the project follows JOYCO conventions), it is the required path for runtime measurement. Do not sprinkle raw `getBoundingClientRect` calls.

What matters for animation handoff:

- **Cached bounds**: `metri.track(el)` registers the element; bounds are cached and kept fresh by one shared throttled ResizeObserver. Reads (`metri.get(el)` document-space, `metri.getBounds(el)` viewport-space) hit the cache, no reflow.
- **Document-space storage**: scroll offset is baked in, so scrolling never invalidates the cache; viewport conversion is a subtraction at read time. For scroll-driven animation prefer document-space reads + `useScrollCallback` (no React re-render per tick) over `useBounds(ref, { space: 'viewport' })` (re-renders every scroll event).
- **React hooks**: `useBounds` (reactive rect), `useScreen` (viewport), `useObserve` (pooled IntersectionObserver visibility), `useTrack` (bounds + visibility + scroll progress 0→1), `useScrollCallback` (scroll without re-render), `DataVisibleSlot` (flips `data-visible`; fine for CSS *state* styling, NEVER as the reveal-animation mechanism, see the selector-flip rule below).
- **Verify the API before use**: check the exact hook names, signatures and options against https://hub.joyco.studio/toolbox/metri before writing code that uses Metri. Do not code against a remembered API; `useObserve` returns `{ visible }` state (no `once` option, no callback form), which shapes the trigger pattern.
- **Custom scroll sources**: if the project uses Lenis or another virtual scroller, Metri must be wired to it via `scrollGetter`/`scrollListener` (or `setScrollHandlers` after mount). Check how the provider is configured before adding scroll-linked animation; reading `window.scrollY` directly in a Lenis project gives wrong values.
- **FLIP with Metri**: First = `metri.getBounds(el)` (cache read). Apply end state. `metri.refresh(el)` (targeted recompute), then Last = `metri.getBounds(el)`. Invert and play with transform. Two measured layouts total, zero interleaved reads/writes.
- **Verifying you did it right**: enable `debug: true` (or `{ interval: 2000 }`) and check `debugSnapshot()`: `getBoundingClientRectCalls` should stay near zero during the animation, cache hits high, `fullRefreshCount` should not spike per interaction. Use `debugReset()` then snapshot around a single interaction to profile it.

When Metri is NOT in the project and the animation needs repeated measurement (scroll-linked, resize-aware, FLIP-heavy), suggest it: `pnpm add @joycostudio/metri`, registry docs at https://hub.joyco.studio/toolbox/metri. For a single one-shot FLIP, plain batched `getBoundingClientRect` is acceptable.

## Selector flips are not an animation mechanism

Driving a reveal through an attribute or class flip plus a CSS transition (`[data-visible="true"] { transition: ... }`, `.is-visible { ... }`) forces the browser to re-match selectors and recalculate style for everything the selector can reach; broad flips invalidate broadly (log 12: "changes body class → every element in the document might be affected"). The convention: initial state ships in the server-rendered output on the `data-motion` selector, and the trigger callback starts the animation imperatively (WAAPI `element.animate` or the target library). `data-motion` targets; it never mechanizes.

## Reduced motion interaction trap

When both CSS `@media (prefers-reduced-motion)` rules and a JS animation library write to the same properties, inline styles from JS win over the media query, silently defeating the accessibility rule (log 13 covers this collision with GSAP; it applies to anime.js and manual code equally). Handle reduced motion at the animation-construction layer (branch in JS), not only in CSS, whenever JS drives the values.
