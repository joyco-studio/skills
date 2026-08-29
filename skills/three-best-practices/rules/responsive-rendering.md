# Responsive rendering

Apply these rules to canvas sizing, cameras, viewports, device pixel density,
orientation changes, and embedded or fullscreen scenes.

## Give CSS and the renderer separate jobs

- Let CSS determine the canvas's displayed size.
- Set the drawing buffer from the measured display size and the selected quality
  scale.
- Do not write CSS dimensions from the render loop or let drawing-buffer changes
  feed back into layout.
- Resize only when integer drawing-buffer dimensions actually change.
- Use a `ResizeObserver` or the project's centralized viewport service for
  container-sized canvases; `window.resize` alone misses layout-driven changes.

## Update every dependent surface together

When the viewport changes, update in one path:

- renderer or framework size;
- camera aspect or orthographic bounds, followed by projection update;
- post-processing composer and pass sizes;
- render targets and depth textures that must match the viewport;
- CSS2D/CSS3D overlays, pointer mapping, and cached coordinate transforms;
- quality-dependent uniforms such as resolution and texel size.

Avoid competing resize effects that observe and write the same dimensions.

## Choose a deliberate pixel-density policy

- Do not assume `devicePixelRatio` is an appropriate render scale.
- Cap or tier density based on target device, scene cost, and visual need.
- Apply the same effective dimensions to post-processing and resolution
  uniforms; mixing CSS pixels and drawing-buffer pixels causes blur or broken
  sampling.
- If adapting quality at runtime, use hysteresis and infrequent changes so the
  renderer does not oscillate between tiers.
- Validate memory and fill-rate impact on high-density phones and laptops.

## Define composition behavior

- Decide whether resize preserves vertical field of view, horizontal framing,
  object scale, or DOM pixel alignment. These are different contracts.
- For orthographic cameras, document the mapping between world units and CSS
  pixels and recompute bounds from that contract.
- Handle zero-sized or hidden containers without producing invalid projection
  matrices or allocating zero/huge render targets.
- Test narrow portrait, wide landscape, browser zoom, orientation change,
  fullscreen transitions, and embedded container resize.

## Avoid per-frame resize churn

The frame loop may check whether cached dimensions changed, but it should not
perform fresh DOM measurement for every dependent object. Measure once,
propagate the result, and update only on change.
