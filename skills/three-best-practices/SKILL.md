---
name: three-best-practices
description: >
  Build, review, debug, and optimize production Three.js experiences. Use for
  Three.js scenes, React Three Fiber, WebGL or WebGPU renderers, GLSL shaders,
  reusable TSL node graphs and typed node contracts, glTF assets,
  post-processing, DOM-to-canvas coordination, render-loop performance,
  first-use GPU hitches, material or texture invalidation, GPU resource cleanup,
  and visual regressions in 3D web work.
  Do not use for generic CSS 3D transforms or static 2D canvas work that does
  not use a Three.js scene graph.
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# Three.js

Build Three.js work that is visually correct, lifecycle-safe, and fast on the
actual target devices.

## Skill structure

This file is the entry point and routing contract. The sibling `rules/`
directory contains the detailed guidance, split by concern so rules can evolve
without turning this index into a second copy of them.

Do not maintain a rule inventory in this file. At the start of every task, scan
the directory itself and treat the discovered Markdown files as the source of
truth. From the directory containing this `SKILL.md`, run:

```sh
rg --files rules | sort
```

Read the discovered filenames before deciding which guidance applies. For a
focused change, open every rule whose concern touches the work. For a new
scene, architecture change, performance pass, or broad review, read all of the
rule documents before acting. When uncertain, read the rule; do not infer its
contents from its filename.

## Workflow

### 1. Inspect the host project

Before proposing code, establish:

- the installed `three` version and whether the app uses vanilla Three.js, a
  framework renderer, or a local engine layer;
- the renderer and target capabilities already in use;
- who owns the canvas, render loop, resize path, asset cache, input handlers,
  and teardown;
- existing conventions for loaders, post-processing, shaders, debugging, and
  tests;
- the target browsers and representative low-power device.

Preserve the project's abstraction level. Do not replace a framework's
lifecycle with an imperative parallel implementation or add a second render
loop, loader cache, or resize system.

### 2. Discover and load the rules

Scan `rules/` as described above. Read all applicable documents before editing.
Resolve overlaps by following the more specific rule, while preserving the
host framework's ownership model.

The rules describe durable engineering constraints, not a substitute for the
installed version's API reference. When an exact method, import path, default,
renderer feature, or addon behavior matters, verify it against the project's
installed source or the matching version of the official Three.js docs.

### 3. Define the visual and performance contract

State what must remain true: composition, camera behavior, asset fidelity,
interaction, loading states, responsive behavior, reduced motion, and frame
budget. For bug fixes, reproduce the issue before changing code. For
optimization, capture a baseline on a representative device before choosing a
fix.

### 4. Implement through existing ownership boundaries

Keep initialization, per-frame mutation, resize, asset ownership, and disposal
explicit. Reuse project primitives and shared resources. Make the smallest
coherent change that satisfies the visual contract, then remove temporary
instrumentation and abandoned branches.

### 5. Validate in the rendered result

Static checks are necessary but insufficient for 3D work. Run the project's
typecheck and tests, then inspect the scene in a browser. Exercise resize,
navigation or remount, asset failure, tab visibility, reduced motion, and the
target input methods. For performance work, compare the same scenario before
and after rather than relying on intuition.

## Decision principles

- Correct ownership beats convenience: one owner for each loop, resource, and
  listener.
- Measure before optimizing and change one bottleneck class at a time.
- Treat GPU memory and shader compilation as lifecycle costs, not invisible
  implementation details.
- Keep resource structure stable in latency-visible paths: prepare and warm
  structure at lifecycle boundaries, then animate values.
- Build shader effects from reusable, typed, composable nodes; keep material,
  resource, and debug ownership outside node math.
- Prefer asset-pipeline fixes over compensating for broken assets at runtime.
- Keep DOM layout reads out of the frame loop; synchronize through cached state.
- Preserve graceful fallback when advanced GPU features are unavailable.

## Completion gate

Before finishing:

- Rescan `rules/` and confirm every applicable rule was followed.
- Verify there is one render-loop owner and a clear teardown path.
- Check resize, pixel density, color pipeline, loading/error states, and input.
- Inspect draw calls, geometry counts, texture memory, shader programs, and
  frame timing when the change can affect performance.
- For first-use hitches, verify whether the transition introduced compilation,
  allocation, upload, binding creation, or synchronous GPU readback.
- For reusable TSL, verify one contract drives compile-time input/output types
  and runtime introspection without duplicating names or value types.
- Confirm navigation or remount does not duplicate canvases, callbacks,
  observers, controls, or GPU resources.
- Report what was tested, the environment used, and any device-dependent risk
  that remains.
