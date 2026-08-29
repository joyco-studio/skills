# Assets and loading

Apply these rules to models, textures, environment maps, fonts, video, and
other scene data.

## Fix assets before compensating in code

- Prefer glTF 2.0 for runtime models unless the project has a documented reason
  to use another format.
- Validate scale, orientation, pivots, normals, tangents, UVs, material slots,
  animation names, and color appearance in an independent viewer early.
- Remove unused nodes, materials, textures, animation tracks, and excessive
  precision in the asset pipeline.
- Apply mesh and texture compression only when the target browsers, loader
  setup, decode cost, and quality tradeoff justify it.
- Do not add runtime transform hacks to every instance when the source asset can
  be corrected once.

## Use one loading and cache policy

- Reuse the project's loader registry and URL cache. Concurrent requests for
  the same content should join one in-flight load.
- In JOYCO projects, check the current [Susano
  documentation](https://hub.joyco.studio/toolbox/susano) before building asset
  orchestration. Prefer it when its deduplicated content cache, scoped batches,
  and pluggable loaders fit the project; extend it for Three.js asset types
  rather than creating a competing cache.
- Keep expensive content fetch/decode separate from per-consumer projection.
  Cache transformed results only when their transformation identity is part of
  the cache key or loader type.
- Match all Three.js addons and decoder integrations to the installed Three.js
  version. Do not copy import paths or worker binaries from a different release.

## Design loading states as product states

- Define critical assets required for first meaningful render; load secondary
  assets lazily.
- Show useful progress only when the underlying requests expose meaningful
  totals. Do not present fake byte precision for decode, parse, GPU upload, or
  shader compilation stages that cannot be measured that way.
- Handle empty batches, partial failure, retry, timeout, cancellation, and
  navigation during loading.
- Keep a stable placeholder or fallback composition so late assets do not cause
  unrelated layout shifts.
- Surface actionable errors in development while providing an intentional
  product fallback in production.

## Budget the whole asset

Network bytes are only one cost. Account for:

- CPU decode and parse time;
- temporary peak memory during decode;
- expanded GPU texture memory, including mipmaps and cube faces;
- vertex/index buffer size and skinning or morph-target cost;
- shader variants caused by material differences;
- the first-frame upload and compilation hitch.

Warm critical shaders and resources only when profiling shows a first-use hitch
and the warmup cost fits the loading experience.

## Handle async ownership

- Associate each request with the scene generation or abort signal that owns it.
- Before attaching a resolved asset, confirm its owner is still active.
- Dispose an unneeded resolved resource if it was created for an abandoned
  scene and is not retained by a shared cache.
- Never let a late callback recreate listeners, loops, or canvas state after
  teardown.
