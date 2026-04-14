# Writing Effects for Suno

A guide to building reusable effect modules on top of `@joycostudio/suno` — filters, reverbs, dynamics, anything that wraps `AudioNode`s. The pattern here is what makes your effect safe to declare at module scope alongside `suno` and `mixer`, survive HMR, and compose cleanly with `mixer.setAmbient` / `play({ output })`.

## The rule: construct pure, attach on demand

Your effect class must be safe to `new` at module scope. That means **zero browser-API access in the constructor** — no `AudioContext`, no `document`, no `window`. Build the node graph in an `attach(suno)` method that gets called from a client-only lifecycle (a React `useEffect`, a DOM-ready handler, etc.).

Why: users want the one-liner:

```ts
export const reverb = new Reverb()
```

at the top of a module, right next to `export const suno = new Suno(...)`. Any browser access in the constructor breaks SSR, blocks module-scope ownership, and forces consumers to write their own `typeof window` guards.

## Canonical pattern

```ts
import type { Suno } from '@joycostudio/suno'

export class Muffle {
  // 1. Buffered params — source of truth pre- and post-attach.
  private _frequency = 20000
  private _enabled = false

  // 2. Live nodes — populated only after `attach()`. Keep the context alongside
  //    so setters can schedule ramps without reaching back through the graph.
  private nodes: { ctx: AudioContext; filter: BiquadFilterNode; head: AudioNode } | null = null

  // 3. Read-only handle to the head of the chain. Null until attached.
  get chain(): AudioNode | null {
    return this.nodes?.head ?? null
  }

  // 4. Build the graph, hand the tail to Suno.
  attach(suno: Suno): void {
    if (this.nodes) return // idempotent
    const ctx = suno.player.audioContext

    const filter = ctx.createBiquadFilter()
    filter.type = 'lowpass'
    filter.frequency.value = this._enabled ? this._frequency : 20000

    suno.effect(filter) // wires filter → masterOutput

    this.nodes = { ctx, filter, head: filter }
  }

  // 5. Reversible teardown — required for HMR.
  detach(): void {
    if (!this.nodes) return
    try { this.nodes.filter.disconnect() } catch {}
    this.nodes = null
  }

  // 6. Setters write the buffered field first, then the live node if attached.
  setFrequency(hz: number): void {
    this._frequency = Math.max(1, hz)
    if (this.nodes) this.nodes.filter.frequency.value = this._frequency
  }

  setEnabled(enabled: boolean): void {
    this._enabled = enabled
    if (!this.nodes) return
    this.ramp(this.nodes.filter.frequency, enabled ? this._frequency : 20000, 0.3)
  }

  private ramp(param: AudioParam, target: number, duration: number): void {
    if (!this.nodes) return
    const { ctx } = this.nodes
    if (ctx.state !== 'running') {
      param.cancelScheduledValues(0)
      param.value = target
      return
    }
    const now = ctx.currentTime
    param.cancelScheduledValues(now)
    param.setValueAtTime(Math.max(param.value, 0.0001), now)
    param.exponentialRampToValueAtTime(Math.max(target, 0.0001), now + duration)
  }
}
```

## Using it

```ts
// lib/audio.ts
import { Suno } from '@joycostudio/suno'
import { Muffle } from './muffle'

export const suno = new Suno({ manifest, mutePersistKey: 'app-muted' })
export const muffle = new Muffle() // SSR-safe, zero browser access
```

Attach from a React effect once on mount:

```tsx
function AudioSetup({ children }) {
  useEffect(() => {
    muffle.attach(suno)
    return () => muffle.detach()
  }, [])
  return children
}
```

Route voices through the head:

```ts
mixer.setAmbient('ambient', { loop: true, output: muffle.chain })
```

Or share the chain across multiple sources:

```ts
suno.get('click').play({ output: muffle.chain })
suno.get('alert').play({ output: muffle.chain })
```

## Six rules

1. **Constructor is pure.** No `suno.player.audioContext`, no `document`, no `window`. If the module can't be imported in Node without throwing, you broke the rule.

2. **Buffer every param in a plain field.** `_frequency`, `_wet`, `_enabled` — whatever your effect exposes. Setters write the buffer first; the live `AudioParam` mirrors it. This lets callers configure the effect before `attach()` runs.

3. **Setters no-op the node writes when unattached.** Guard live-node access with `if (this.nodes)`. Buffered-field writes still happen so the values seed correctly when `attach()` runs.

4. **`attach(suno)` is idempotent.** Return early if `this.nodes` is already set. Users call attach from React effects that may fire twice under StrictMode or HMR.

5. **Hand the tail to `suno.effect(lastNode)`.** Don't manually `connect(suno.player.masterOutput)` — letting Suno own the last hop keeps your effect composable and symmetric with other effect modules.

6. **`detach()` disconnects every node you built, then nulls the handle.** Don't try to reset params — the nodes are going away. Wrap each `.disconnect()` in `try/catch`; Web Audio throws if the node was already unwired downstream.

## Parallel graphs

`suno.effect(...)` chains nodes in sequence. For parallel topologies (dry/wet reverb, sidechain, parallel EQ), wire the graph manually inside `attach()` and hand only the final tail to `suno.effect`:

```ts
attach(suno: Suno) {
  const ctx = suno.player.audioContext
  const head = ctx.createGain()
  const dry = ctx.createGain()
  const wet = ctx.createGain()
  const convolver = ctx.createConvolver()
  convolver.buffer = buildIR(ctx)
  const tail = ctx.createGain()

  head.connect(dry)
  head.connect(convolver)
  convolver.connect(wet)
  dry.connect(tail)
  wet.connect(tail)

  suno.effect(tail) // tail → masterOutput

  this.nodes = { ctx, head, tail, dry, wet, convolver }
}
```

`head` is what you return from `get chain()`; it's the node callers pass as `output` on `play()` / `setAmbient`.

## Suspended-state construction is fine

You can `attach()` before the user has interacted with the page. The `AudioContext` will be in `suspended` state, nodes will build and wire up, and no audio flows. Your ramp helpers should detect `ctx.state !== 'running'` and write param values directly — the timeline isn't advancing yet, so a scheduled ramp would land at an unpredictable moment. Once Suno unlocks, audio flows through your pre-built graph with no reattach needed.

This means you don't have to gate `attach()` on `onUnlock` — mount-time attachment works and avoids a class of "my ambient won't play because the chain wasn't ready when the effect fired" bugs.

## Don't globalize what should be per-voice

Suno's `play({ output })` and `mixer.setAmbient({ output })` let any voice route through your effect. **Prefer this over inserting your chain between `masterOutput` and `destination`.** An effect that globally muffles every sound — hover clicks, form-submit dings, ambient — is almost always the wrong design.

Expose your `chain` and let callers pick the voices that get decorated:

```ts
// Ambient through the muffle
mixer.setAmbient('bed', { output: muffle.chain })

// Voice lines through a separate telephone-EQ effect
suno.get('vo-1').play({ output: phoneEQ.chain })

// One-shots stay dry
suno.get('click').play()
```

## Checklist

- [ ] Constructor touches no browser APIs.
- [ ] Every param has a buffered field.
- [ ] Every setter writes the field first, then optionally the live node.
- [ ] `attach()` is idempotent and side-effect-free before it's called.
- [ ] `detach()` disconnects every owned node and nulls the handle.
- [ ] `chain` (or equivalent head accessor) returns `null` pre-attach.
- [ ] Ramp helpers check `ctx.state === 'running'` before scheduling.
- [ ] Graph tail is handed to `suno.effect()`, not manually connected to `masterOutput`.
