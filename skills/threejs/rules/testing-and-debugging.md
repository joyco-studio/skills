# Testing and debugging

Apply these rules to new features, bug fixes, performance work, and reviews.

## Reproduce before changing code

- Record the exact route, viewport, device scale, input sequence, assets, and
  browser needed to expose the issue.
- Separate visual correctness, interaction, lifecycle, CPU performance, GPU
  performance, and memory hypotheses.
- Change one variable at a time. Disable passes, lights, shadows, groups, or
  materials selectively to isolate the failing layer.
- Prefer temporary instrumentation with a clear removal point over speculative
  rewrites.

## Use the scene as evidence

- Inspect renderer diagnostics for calls, triangles, points, lines, textures,
  geometries, and programs as supported by the installed version.
- Use browser frame recordings to distinguish JavaScript/layout work from GPU or
  compositor pressure.
- Check shader compile errors and WebGL/WebGPU validation output.
- Add temporary axes, grids, bounds, normals, camera helpers, wireframes, or
  flat debug materials to verify geometry and coordinate assumptions.
- Capture a stable screenshot or video reference before fidelity-sensitive
  changes.

## Test lifecycle failure modes

- Repeatedly mount, unmount, and revisit the scene.
- Resize while assets load and navigate away before they resolve.
- Hide and restore the tab, then verify time and animation do not jump.
- Trigger missing, corrupt, slow, and partially failing assets.
- Lose focus or cancel a pointer during drag.
- When feasible, exercise context loss and recovery or confirm the product's
  explicit fallback behavior.

## Validate the user matrix

At minimum, cover the supported combinations that materially change behavior:

- pointer and touch input;
- keyboard and semantic DOM alternatives for essential actions;
- reduced motion;
- low and high device pixel density;
- narrow and wide layouts;
- one representative constrained GPU/device;
- the supported browser renderer paths.

Do not claim broad GPU or browser correctness from a single desktop browser.

## Automate stable contracts

- Unit-test math, state transitions, asset transforms, and pure coordinate
  conversions outside the renderer.
- Integration-test initialization, resize propagation, loading/error states,
  and teardown counts where the project harness supports them.
- Use visual regression tests only with controlled camera, assets, timing,
  random seeds, viewport, and renderer settings.
- Use tolerances appropriate to GPU raster differences; test meaningful regions
  or invariants rather than demanding impossible bit-identical output.

## Hand off evidence

Report commands run, browser/device tested, the scenario used, measured before
and after values for performance work, and any renderer- or device-specific
risk left untested.
