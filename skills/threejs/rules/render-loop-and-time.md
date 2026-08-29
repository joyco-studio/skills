# Render loop and time

Apply these rules to animation, simulation, controls, scroll-driven scenes, and
on-demand rendering.

## Keep one scheduler

- Use the render loop already owned by the host framework or engine.
- Do not nest `requestAnimationFrame`, renderer animation callbacks, framework
  frame hooks, or animation-library tickers for the same scene.
- When vanilla Three.js owns the loop, prefer the renderer's supported animation
  loop facility when XR compatibility matters. Verify the exact API against the
  installed version.
- Store the callback identity needed for cancellation and stop scheduling
  before disposing scene resources.

## Use time, not frame count

- Express animation in seconds and make movement independent of refresh rate.
- Use elapsed time for deterministic presentation animation and delta time for
  incremental simulation.
- Clamp or reset large deltas after a hidden tab, debugger pause, route
  transition, or suspended device. A resumed frame must not launch a simulation
  across seconds of missing time.
- Fixed-step physics should accumulate real time, execute bounded fixed steps,
  and drop excess backlog rather than spiraling indefinitely.

## Render only when useful

- Prefer demand rendering for static scenes and product viewers. Invalidate on
  camera, control, asset, animation, resize, or state changes.
- Pause nonessential work when the document is hidden or the scene is outside
  its meaningful viewport.
- Continue rendering only while damping, animation, video textures, simulation,
  or another time-dependent input actually changes the frame.
- Avoid a second idle polling loop merely to discover whether rendering is
  needed; update the invalidation signal at the source of change.

## Keep the hot path allocation-free

- Allocate reusable vectors, matrices, quaternions, colors, raycasters, and
  arrays outside the frame callback.
- Avoid object spreads, mapped arrays, string construction, and new closures in
  per-object frame loops.
- Update transforms in batches where possible. Do not repeatedly traverse the
  entire scene when the affected set is already known.
- Mark buffer attributes, instance matrices, and other GPU data dirty only when
  their contents changed.

## Order each frame deliberately

Use one predictable phase order, adapted to the host project:

1. Sample input and external state.
2. Advance clocks, controls, animation, and simulation.
3. Apply scene transforms and camera changes.
4. Update dependent uniforms or derived matrices.
5. Render all required passes.
6. Record diagnostics outside production builds.

Do not read DOM layout between scene updates. Layout measurement belongs in a
separate cached measurement path.
