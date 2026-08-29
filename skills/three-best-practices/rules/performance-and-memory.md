# Performance and memory

Apply these rules to every nontrivial scene and whenever frame rate, heat,
startup time, or memory is a concern.

## Measure the right bottleneck

Capture a repeatable scenario on a representative device. Separate:

- main-thread JavaScript, style, layout, and garbage collection;
- render submission and draw-call overhead;
- vertex/fragment shader and fill-rate cost;
- texture, buffer, and render-target memory;
- asset decode, shader compilation, and first-use upload;
- compositor work around the canvas and surrounding DOM.

Use browser performance tools, the renderer's diagnostics, and a GPU profiler
when available. Average FPS alone hides spikes and cannot identify the cause.
Track frame-time percentiles and visible hitches.

## Control resolution and fill rate

- Size the drawing buffer from the canvas's displayed size, not blindly from
  the window.
- Cap effective pixel density according to the product's quality policy; raw
  device pixel ratio is often too expensive on mobile hardware.
- Avoid resizing the drawing buffer unless its integer dimensions changed.
- Downsample expensive post-processing and offscreen passes when fidelity
  permits.
- Watch transparent overdraw, full-screen shader passes, large particles, and
  shadow-map fill cost.

## Reduce submission cost deliberately

- Reuse geometry and materials.
- Instance repeated meshes when their material and geometry contract allows it.
- Merge only genuinely static compatible geometry; do not sacrifice useful
  culling or editing boundaries blindly.
- Use level of detail, frustum culling, and visibility gating for expensive
  distant or offscreen content.
- Reduce material and light combinations that create unnecessary shader
  variants.
- Keep skinned meshes, morph targets, dynamic buffers, and shadow casters within
  an explicit budget.

## Avoid hot-path churn

- Preallocate math objects and typed arrays used each frame.
- Do not recreate geometries, materials, textures, uniform containers, or
  post-processing chains during animation.
- Update only the changed range of dynamic buffers when supported by the
  installed version and usage pattern.
- Avoid traversing, sorting, or querying the full graph every frame when a
  stable affected set can be maintained.
- Keep DOM measurement and framework state updates out of the render loop.

## Watch memory as dimensions, not file size

- Estimate texture memory from decoded dimensions, format, mip levels, layers,
  and cube faces—not compressed transfer bytes.
- Use appropriately sized textures and power-of-two dimensions only where the
  renderer or intended sampling features benefit from them.
- Dispose replaced resources and verify stable counts over repeated lifecycle
  tests.
- Treat render targets, shadow maps, environment preprocessing, video textures,
  and post-processing buffers as part of the same GPU budget.

## Change one cost center at a time

Record the baseline, apply the smallest plausible improvement, rerun the same
scenario, and compare. Keep a visual reference so an optimization cannot
silently reduce correctness. Revert complexity that does not produce a useful
measured win.
