# TSL node design

Apply these rules when creating reusable Three.js Shading Language graphs or
tooling that inspects and edits their inputs.

## Prefer composable semantic nodes

Build effects by chaining nodes with explicit inputs and outputs instead of
writing one product-specific TSL program. A useful node owns one coherent
transformation—such as coordinate mapping, field shaping, masking, grading, or
blending—with a real name, invariants, reuse potential, or independent test
value.

Do not extract every arithmetic expression. Keep one-line material wiring and
local coercions inline. Product-specific nodes may orchestrate smaller nodes,
but reusable math should remain independently callable.

A node module owns its contract, shader math, numerical guards, and documented
coordinate/value assumptions. Its consumer owns uniforms, textures, materials,
render state, loading, debug UI, and disposal. Do not let reusable shader math
load assets, subscribe to product state, create materials, or own lifecycle.

Prefer explicit coordinates, time, values, masks, and resources over hidden
inputs. Use a built-in TSL global only when the effect is intrinsically coupled
to it, and document that coupling.

## Use one contract for types and runtime

Declare each node once with a stable name, named input types, and an output type:

```ts
const effect = defineNode(
  {
    name: "effect",
    inputs: {
      value: "vec3",
      amount: "float",
      noiseMap: "texture",
      coordinates: "vec2",
    },
    output: "vec3",
  },
  ({ value, amount, noiseMap, coordinates }) => {
    const noise = noiseMap.sample(coordinates).rgb;
    return mix(value, noise, amount);
  },
);
```

Keep the node's defining math inside this callback. Calling other declared,
reusable nodes is composition; hiding the whole effect behind an unrelated
helper only moves the implementation away from its contract.

This declaration is authoritative. Use it to infer the implementation inputs,
check the output, type consumer input bags, and expose runtime metadata. Do not
maintain parallel TypeScript interfaces, schemas, debug lists, or gallery
descriptions.

The project-local node factory should:

- preserve type strings as literals with const generics or equivalent;
- map scalar, vector, and matrix strings to their matching `Node<T>` types;
- preserve specialized resource APIs—for example, `texture` maps to a texture
  node rather than a generic node;
- centralize that type mapping so new supported types have one registration
  point;
- keep the declared output authoritative with `NoInfer` or an equivalent
  boundary;
- attach a copied, frozen runtime definition to the callable node.

Keep type facts in the definition, not live resources or mutable values. Treat
names as stable identifiers once tooling, persistence, or telemetry uses them.

Derive consumer types from `typeof node`. Use a strict type when one owner
provides the complete input bag and a partial type when call-site composition
supplies the rest. Prefer `satisfies` so the bag is checked without widening its
inferred properties:

```ts
const inputs = {
  amount: uniform(0.5),
  noiseMap: texture(noiseTexture),
} satisfies NodeInputs<typeof effect>;
```

The final node call must still satisfy the complete contract.

## Make tooling contract-driven

Runtime metadata lets generic tools display contracts, build node galleries,
validate connector configuration, generate fixtures, and inspect nodes without
importing product-specific material code.

A debug connector should receive the defined node, the consumer-owned input
bag, and presentation metadata. It reads names and value types from the node
definition; the connector configuration adds only UI concerns such as range,
step, label, or rebuild behavior.

Type connector configuration from the node contract so controls cannot target
unknown inputs or apply vector controls to scalars. At runtime, verify editable
inputs are uniform nodes. Textures, attributes, varyings, and computed nodes are
valid inputs but are not automatically editable controls.

Live controls should mutate existing uniform `.value` fields declaratively.
Ordinary numeric, color, boolean, or vector changes must not rebuild the graph,
replace materials, or set `material.needsUpdate`.

Keep slider ranges and labels outside the core contract unless they are genuine
domain invariants shared by every consumer.

## Preserve composition and runtime stability

- Construct the bounded graph once and retain stable node identities.
- Animate through uniforms, attributes, UVs, and existing value nodes.
- Prebuild meaningfully different algorithms or switch them at a controlled
  lifecycle boundary; do not replace branches per frame.
- Keep resource slots stable when textures change during play.
- Return the narrowest useful output for chaining: expose a scalar mask when
  callers may color it differently.
- Name inputs by semantic role, not by one UI or call site.
- Document coordinate spaces, units, ranges, normalization, alpha behavior, and
  HDR/color semantics.

Composability does not justify an unbounded graph of tiny wrappers. Keep the
final graph understandable, bounded, and sensible as shader work.

## Validate the contract

- Compile-time tests reject missing or unknown inputs, wrong node types, and a
  return value that conflicts with the declared output.
- Runtime tests verify the exposed name, inputs, output, and frozen metadata.
- Unit tests cover numerical edge cases and composition outside a material when
  practical.
- Preview or gallery tests prove reuse outside the original feature.
- Connector tests verify supported uniforms become controls and non-editable
  resources or computed nodes fail clearly.

Before finishing, confirm the node is a reusable semantic effect, one contract
drives static and runtime behavior, consumers retain lifecycle ownership, and
debugging can change values without changing graph structure.
