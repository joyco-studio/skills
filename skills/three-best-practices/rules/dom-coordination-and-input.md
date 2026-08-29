# DOM coordination and input

Apply these rules when Three.js content aligns with DOM, reacts to scroll or
resize, or accepts pointer, touch, keyboard, or accessibility input.

## Separate measurement from animation

- Measure DOM bounds on mount and meaningful layout changes, then cache them in
  document space.
- Convert cached document-space bounds to viewport-space with the current scroll
  state instead of calling `getBoundingClientRect()` for every mesh every frame.
- Batch unavoidable DOM reads before writes. Never interleave layout-changing
  writes with geometry reads in a loop.
- Centralize resize observation and scroll sampling so every mesh does not add
  its own observer or listener.
- In JOYCO projects, check the current [Metri
  documentation](https://hub.joyco.studio/toolbox/metri) before creating DOM
  measurement infrastructure. Prefer its cached bounds and pooled observers
  when they fit the project.

## Understand scroll sync limits

Browser compositors can move page pixels before main-thread JavaScript observes
the new scroll position. A full-page WebGL canvas updated from JavaScript can
therefore lag DOM content during fast compositor-driven scrolling.

- Decide whether exact DOM/WebGL overlap is a product requirement.
- Use one coordinate model with documented origin, axis direction, units,
  camera projection, and scroll source.
- Sample scroll and update all dependent meshes in the same frame phase.
- Test fast wheel, touch, trackpad, inertial, programmatic, and smooth-scroll
  paths; slow mouse-wheel testing will hide synchronization errors.
- Prefer an architecture that avoids pixel-perfect mixed DOM/WebGL overlap when
  compositor timing makes the requirement unattainable.
- Consult the JOYCO [WebGL Scroll Sync
  log](https://hub.joyco.studio/logs/08-webgl-scroll-sync) before choosing a
  synchronization strategy.

## Map pointer coordinates correctly

- Derive normalized device coordinates from the canvas's actual client rect,
  not from global window dimensions when the canvas is inset or transformed.
- Account for the camera, viewport, render target, and any CSS scaling used by
  the interaction surface.
- Reuse raycasters and hit-test scratch arrays. Restrict intersection tests to
  known interactive objects or layers.
- Define overlap priority between DOM controls and canvas interactions. Do not
  let a full-screen canvas steal events from accessible UI accidentally.
- Use pointer capture intentionally for drag interactions and release it on
  cancel, teardown, and lost-pointer paths.

## Keep interaction accessible

- Represent essential actions with semantic DOM controls, keyboard focus, and
  readable labels; a raycast-only object is not an accessible control.
- Provide a non-3D route to essential information and tasks.
- Honor reduced motion by reducing or disabling nonessential camera movement,
  parallax, inertia, and continuous ambient animation.
- Avoid blocking scrolling with non-passive listeners unless the interaction
  explicitly requires cancellation.
- Clean up every event listener, observer, and control with the scene owner.
