# GPU invalidation and hot-path stability

Apply these rules when a scene hitches the first time an object appears, a
texture changes, an effect starts, or a supposedly small runtime mutation
causes a slow later frame.

## Use the right mental model

The central distinction is between interpolation and invalidation:

- **Interpolation** changes values while resource structure stays fixed:
  transforms, uniforms, opacity, UVs, colors, or bounded instance attributes.
- **Invalidation** tells the renderer that a cached GPU representation may no
  longer be reusable. The renderer may have to compile, allocate, upload,
  convert, bind, or rebuild before drawing correctly.

Interpolation is not automatically cheap. A callback that creates resources,
changes shader features, grows a buffer, or raises an invalidation flag still
crosses the expensive boundary.

The governing rule is:

> Keep resource structure stable in hot paths. Prepare structure early, then
> animate values.

## Expect deferred cost

The CPU and GPU work asynchronously. A JavaScript assignment often records a
version or state change without performing the expensive operation immediately.
On a later render, Three.js builds the render list, compares versions and cache
keys, prepares backend resources, submits commands, and only then can the GPU
execute them.

This is why animation code can look cheap in a CPU profile while the next frame
stalls. A completed fetch or JavaScript cache hit also does not prove that a
texture, program, pipeline, buffer, or binding is resident and ready on the GPU.

## Recognize the three expensive boundaries

### Compilation and pipeline creation

Materials, shaders, and TSL graphs must become executable GPU programs or render
pipelines. Their effective variants may depend on:

- material or node-graph type;
- available texture and sampler features;
- lights, shadows, fog, skinning, morph targets, and vertex colors;
- transparency, blending, depth, stencil, sidedness, and clipping;
- render-target formats, multisampling, and output configuration.

The first draw of a new combination may generate shader source, compile, link,
or create a pipeline. Mobile drivers often make this first-use work visible.

- Construct every bounded material graph and feature combination before
  interactive rendering.
- Prefer uniforms over changing defines, replacing node graphs, or toggling
  shader features during play.
- Keep sampler slots present for the material lifetime when a texture will
  appear later.
- After the real lights, environment, targets, and materials exist, use the
  installed renderer's supported asynchronous compilation or a representative
  warmup render.
- Do not compile every frame. Warmup moves bounded work; it does not make
  unbounded variant creation safe.

### Allocation and upload

Decoded assets are not necessarily GPU-resident. Textures, mip levels, geometry
buffers, instance data, render targets, depth buffers, samplers, and bindings
may be allocated or uploaded only when first needed.

- Finalize texture format, color space, filtering, wrapping, flipping, and mip
  policy before first upload.
- Initialize or render representative textures before a latency-sensitive
  reveal when profiling shows first-upload stalls.
- Allocate geometry, instance buffers, render targets, and pools at a bounded
  capacity before play.
- Reuse resources instead of constructing and disposing them during gameplay.
- Keep legitimate dynamic uploads small, explicit, and bounded.
- Treat resize or format changes as controlled lifecycle events because they
  can reallocate render targets and depth resources.

A texture replacement is predictable only when the sampler slot already exists,
the replacement is resident, and the backend has already encountered the
required binding combination.

### CPU-to-GPU synchronization

Most GPU work is queued asynchronously. A synchronization point forces the CPU
or GPU to wait and can expose all preceding queued work as one visible stall.

Common hazards include:

- synchronous render-target or canvas readback;
- screenshots during an active interactive frame;
- querying GPU results before they are available;
- disposing or reallocating resources still in flight;
- memory pressure that forces eviction or driver cleanup;
- compiling or uploading a prerequisite immediately before its first draw.

Keep synchronous readback out of hot paths. Prefer an installed asynchronous
readback API, delayed result consumption, or a GPU-side solution. Schedule
unavoidable capture, rebuild, or resize behind a loading state, pause, or
controlled transition.

## Define hot paths by visible latency

A hot path is any path whose latency is visible while the user expects
continuous response—not only a callback that runs every frame. It includes:

- animation, simulation, render, and pre-render callbacks;
- input handlers that affect the current frame;
- procedural spawning, recycling, collisions, scoring, and transition effects;
- the first appearance of an object or effect during play.

A task that runs once is still a hot-path bug when that one execution misses the
frame budget.

Keep the following outside those paths:

- constructing or disposing meshes, materials, geometry, textures, or targets;
- changing shader topology or material feature presence;
- invalidating a material for an ordinary visual transition;
- invalidating unchanged static texture content as a generic refresh;
- changing texture dimensions, format, filtering, wrapping, or mip policy;
- growing typed arrays or GPU buffer capacity;
- compiling, warming, or performing synchronous readback;
- allocating large temporary arrays or per-particle objects.

## Classify runtime changes before implementing

| Planned change | Expected treatment |
| --- | --- |
| Transform, camera, uniform, color, opacity | Mutate the existing value |
| Sprite frame or UV window | Update a small UV value or bounded buffer |
| Instance transforms or particle state | Update preallocated instance buffers |
| Texture choice | Bind a resident texture to an existing sampler slot |
| True texture-content change | Use a dedicated dynamic-texture path |
| Shader feature or material graph | Prebuild a variant or use a uniform branch |
| Geometry layout or capacity | Preallocate or pool; do not grow during play |
| Render-target size or format | Allocate at a controlled lifecycle boundary |
| GPU data needed by the CPU | Read asynchronously and consume later |

## Interpret `needsUpdate` by resource type

Do not ban every `needsUpdate`. Identify the invalidated layer:

- `Material.needsUpdate` may invalidate compiled shader or pipeline state.
- `Texture.needsUpdate` schedules source upload and texture configuration.
- `BufferAttribute.needsUpdate` schedules changed buffer data for upload.
- `InstancedMesh.instanceMatrix.needsUpdate` schedules instance data for upload.

The buffer cases are normal runtime mechanisms when storage already exists and
the changed range is bounded. They still consume bandwidth: raise the flag only
after visible data changes and use supported update ranges where appropriate.
Material and static-texture invalidation are planned lifecycle events, not
repaint buttons.

## Prefer stable workaround patterns

### Replace feature toggles with values

Construct the final feature shape before play. Keep a compatible placeholder in
any sampler slot that will be used later, then animate opacity, a blend uniform,
or another numeric value. For TSL or custom shader graphs, keep nodes and graph
topology stable and change their values or uniforms.

### Crossfade fixed resources intentionally

When a real texture crossfade is required, build both sampler slots into one
bounded shader variant, make both textures resident, then animate one blend
uniform. This trades extra sampling cost for predictable frame time. A discrete
swap should keep one stable sampler instead of paying for two samples forever.

### Pool variable populations

For particles, projectiles, enemies, or recycled scenery:

- allocate a fixed or bounded slot count;
- reuse meshes or instances;
- hide or deactivate unused slots;
- update existing transforms and attributes;
- define overflow behavior instead of allocating reactively.

Pooling controls allocation. Instancing is a separate draw-call decision; keep
depth, transparency, culling, and ordering semantics correct when choosing the
batch boundary.

### Isolate truly dynamic textures

Canvas, video, procedural data, and streaming sources legitimately change pixel
content. Use the version-appropriate dynamic texture type, keep dimensions and
format stable, update only when the source changes, and cap resolution and
frequency to the upload budget. Prefer GPU-side generation or render targets
when the data already lives on the GPU.

## Warm representative usage

A useful warmup reproduces later resource conditions without advancing product
state:

1. Construct the final scene and render pipeline.
2. Temporarily expose all bounded lazy materials, textures, and effect variants.
3. Initialize textures and render targets using APIs supported by the installed
   renderer and version.
4. Compile asynchronously where supported.
5. Render representative frames when first draw performs additional work.
6. Restore the exact initial state, then reveal the experience.

Warmup is incomplete if it visits only the initial visible state. It is not a
substitute for stable runtime design; a continually growing variant set cannot
be warmed exhaustively.

## Diagnose first-use hitches

Ask:

1. Did this frame encounter a program, pipeline, texture, geometry, target, or
   binding combination for the first time?
2. Did material or texture versions, object counts, programs, or renderer memory
   counters change during play?
3. Did the latency-visible path construct, dispose, clone, resize, grow, compile,
   warm, read back, or set `needsUpdate`?
4. Was an asset only fetched and decoded, or had it also been uploaded and drawn?
5. Did a previously absent feature become present, such as an empty sampler slot
   receiving a texture?
6. Did the CPU synchronously request data produced by the GPU?
7. Can fixed resources plus uniforms, UVs, instances, or pooled activation
   express the same effect?

Instrument versions and resource counts around the transition. Compare the
first and second occurrence: a slow first use followed by fast repeats strongly
suggests lazy compilation, upload, allocation, or binding creation.
