# Color, lighting, and post-processing

Apply these rules whenever visual fidelity, textures, lights, environment maps,
render targets, or post-processing are involved.

## Keep the color contract explicit

- Lighting and shader math belong in a linear working space.
- Annotate color textures such as base-color and emissive images with the
  appropriate color space. Keep data textures such as normals, roughness,
  metalness, masks, and most lookup data untagged as color data.
- Treat HDR environment and light data according to the encoding and format
  that actually produced it; do not mark every image as sRGB.
- Ensure the final displayed output is converted once to the intended display
  color space.
- When post-processing, use the version-appropriate output conversion at the
  end of the chain. Do not encode once in a material and again in the final pass.

Three.js defaults and names have changed across releases. Inspect the installed
version before setting renderer color properties or assuming a default.

## Diagnose appearance in pipeline order

When a scene looks washed out, too dark, or unlike the DCC source, check:

1. Texture color-space annotations.
2. Working-space assumptions in custom shader math.
3. Environment intensity, exposure, and light units.
4. Tone mapping and final output conversion.
5. Browser/CSS treatment of the canvas and the source asset itself.

Do not tune arbitrary light intensities to hide a double conversion or wrongly
tagged texture.

## Keep lighting intentional

- Use physically coherent materials and environment lighting when asset
  fidelity depends on PBR.
- Process environment maps through the installed Three.js workflow appropriate
  to the renderer and material system.
- Share environment resources and dispose temporary processing resources.
- Keep shadow-casting lights and shadow maps to the smallest count and resolution
  that achieve the visual goal.
- Tighten shadow camera bounds and update static shadows only when needed.
- Avoid multiplying material variants solely to make minor intensity changes
  that could be data or instance-driven.

## Treat post-processing as a render pipeline

- Keep pass order deliberate and documented when order changes the image.
- Size every composer and render target with the same explicit viewport and
  pixel-density policy as the renderer.
- Resize and dispose pass-owned targets when the viewport or lifecycle changes.
- Avoid full-resolution passes for effects that tolerate downsampling.
- Validate alpha, premultiplication, depth, multisampling, and color conversion
  across the complete chain, not one pass in isolation.
- Provide a lower-cost path for constrained devices and reduced-motion users
  when an effect is temporal or expensive.

## Validate with references

Compare against a known reference under controlled camera, exposure, light, and
background conditions. A screenshot taken with different tone mapping or
environment lighting is not a useful fidelity baseline.
