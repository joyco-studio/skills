# Project architecture

Apply these rules whenever creating or restructuring a Three.js feature.

## Respect the host

- Read the manifest and lockfile to identify the exact Three.js and framework
  versions before using an API from memory.
- Follow existing imports and addon resolution. Keep core and addons on the
  same installed Three.js version; duplicate Three.js instances can break
  identity checks and inflate the bundle.
- If the project uses React Three Fiber or another renderer, let that renderer
  own reconciliation, the render loop, events, and disposal unless its escape
  hatch is intentionally being used.
- Reuse established scene services, loaders, material factories, shader
  utilities, and quality settings. Do not introduce a parallel mini-engine for
  a local feature.

## Make ownership visible

Each subsystem needs one clear owner:

- Canvas and renderer creation
- Scene and camera lifetime
- Frame scheduling and time
- Resize and pixel-density policy
- Asset loading and caching
- Pointer, keyboard, and scroll subscriptions
- GPU resources and teardown

Prefer a small scene/controller boundary with explicit `mount`, `resize`,
`update`, and `dispose` responsibilities over scattered module-level state.
Module singletons are acceptable only when the application intentionally owns
them for its full lifetime.

## Keep helper logic out of entities

- Do not mix an entity's core functionality with business logic owned by a
  helper. Keep that boundary explicit.
- A helper should be removable by deleting its file and references, without
  having to extract helper-specific behavior from the entities it used.
- Keep important entity behavior close to the entity so scanning the code
  reveals the core logic without helper concerns obscuring it.

## Separate stable state from frame state

- Create geometry, materials, loaders, render targets, and controls outside the
  hot loop.
- Keep per-frame work to mutations of already-owned objects and preallocated
  scratch values.
- Keep product state in the host application's state model. Derive the minimal
  numeric state needed by the renderer instead of duplicating business state in
  the scene graph.
- Do not trigger framework reconciliation on every frame. In React Three Fiber,
  mutate refs inside `useFrame` for transient animation state and reserve React
  state for meaningful UI state.

## Design for capability differences

- Detect required GPU or browser capabilities before selecting an advanced
  renderer or effect.
- Define a deliberate fallback: lower quality, simpler material, static image,
  or clear unsupported state.
- Keep renderer-specific code behind a narrow boundary when the project may
  switch between WebGL and WebGPU paths.
- Do not silently remove accessibility-critical content when 3D is unavailable.

## Keep the scene inspectable

- Give important objects stable names or metadata meaningful to the domain.
- Group by lifecycle and responsibility, not merely by visual proximity.
- Keep debug helpers behind a development flag and remove or disable them in
  production.
- Comment coordinate-system or unit conventions when they differ from standard
  Three.js expectations.
