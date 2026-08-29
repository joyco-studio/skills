# Resources and disposal

Apply these rules whenever a scene, route, model, effect, or material can be
replaced or unmounted.

## Treat removal and disposal as different operations

Removing an object from the scene graph does not release its GPU allocations.
Track and dispose owned resources explicitly, including:

- geometries and instanced geometry;
- materials, including every material in an array;
- textures referenced by material properties and shader uniforms;
- render targets, depth textures, and post-processing passes;
- controls, helpers, loaders with abortable work, and animation mixers;
- renderer-owned resources when the renderer itself is being destroyed.

Use the installed APIs to determine which resource types expose `dispose()`.
Do not assume a generic scene traversal finds every resource: environment maps,
backgrounds, uniform textures, render targets, skeleton data, and shared caches
may live outside ordinary mesh properties.

## Dispose by ownership

- The component that creates a resource owns its disposal unless ownership is
  explicitly transferred.
- Never dispose a shared geometry, material, texture, or environment while
  another scene consumer still references it.
- Use reference counting, cache eviction, or an application-lifetime owner for
  shared resources. Document which policy applies.
- Keep a resource registry when a feature creates a dynamic graph that cannot
  be reliably enumerated at teardown.
- Remove the object from its parent before releasing resources that should no
  longer render.

## Tear down the entire runtime boundary

On unmount or replacement:

1. Stop the render loop and time-dependent callbacks.
2. Abort or ignore stale async loads.
3. Remove event listeners, observers, controls, and debug panels.
4. Detach scene objects and dispose resources owned by the feature.
5. Dispose passes, targets, caches, and the renderer when locally owned.
6. Remove the canvas only if this feature created it.
7. Clear references that otherwise keep the graph reachable.

Framework auto-disposal is useful but not universal. Verify escape hatches,
primitive objects, shared resources, cached assets, and custom passes rather
than assuming the framework covers them.

## Prove teardown works

- Mount and unmount the feature repeatedly while watching renderer diagnostics,
  browser memory, listeners, and canvas count.
- Allow a load to finish after unmount and verify it cannot mutate or resurrect
  the old scene.
- Re-enter the route and confirm there is still one loop, one canvas, and one
  set of handlers.
- Expect some engine caches to remain stable across frames; investigate values
  that grow across identical mount/unmount cycles.
