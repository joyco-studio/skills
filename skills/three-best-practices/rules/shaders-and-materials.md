# Shaders and materials

Apply these rules to custom shader code, material extensions, procedural
effects, GPU simulation, and material variant work.

## Start from the smallest abstraction that fits

- Use built-in materials when they satisfy the visual contract.
- Extend an existing material only when integration with its lighting, shadows,
  fog, skinning, morph targets, clipping, or renderer features is required.
- Use a fully custom shader when owning that entire contract is intentional.
- Follow the project's existing shader toolchain and preprocessing. Do not add a
  second include, transpilation, or uniform abstraction casually.

## Keep shader contracts explicit

- Document coordinate spaces for attributes, varyings, normals, view vectors,
  depth, and positions.
- Perform lighting and color math in the pipeline's linear working space; apply
  final output conversion exactly once.
- Keep CPU uniform types aligned with GLSL/WGSL declarations and update
  frequencies.
- Distinguish compile-time variants from runtime uniforms. Use variants only
  when they remove meaningful work or change required program structure.
- Initialize every path and handle precision, division by zero, degenerate
  normals, invalid depth, and out-of-range texture coordinates deliberately.

## Control compilation and variants

- Avoid generating unique material programs from values that can remain
  uniforms or instance data.
- Keep preprocessor defines stable during animation; changing them can trigger
  recompilation.
- When patching a built-in shader, include a stable cache key and tests for the
  installed Three.js version. Internal shader chunks are not a permanent API.
- Compile or warm critical variants only when measurements show first-use
  stalls, and include that work in the loading budget.
- Surface compile and link failures in development rather than leaving a black
  object with no context.

## Design for GPU cost

- Move invariant work out of the fragment stage or precompute it when practical.
- Minimize texture samples, divergent branches, high-frequency noise, and
  overdraw in large screen regions.
- Use the lowest precision that preserves correctness on target hardware, but
  verify visible banding and mobile behavior.
- Bound loops and ray-marching steps, expose a quality tier, and add early exits
  where they preserve the effect.
- Remember that transparent fragments, discarded fragments, and fullscreen
  passes can remain expensive even with very little geometry.

## Manage uniforms and textures by lifecycle

- Reuse uniform containers and math values instead of replacing objects every
  frame.
- Update only values that changed.
- Dispose textures, targets, and custom material resources according to their
  owner; shader uniforms can hide resources from naive scene traversals.
- When cloning a material, decide explicitly which uniforms and textures are
  shared and which are independent.

## Validate beyond one machine

Test shader compilation, color, precision, alpha, depth, and performance on at
least one constrained target. Provide a simpler fallback when the required
features or budget are unavailable.
