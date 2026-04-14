---
name: webaudio
description: >
  Add sound effects, UI audio, and ambient sound to a web app using the
  @joycostudio/suno library. Use when the user wants to play audio on
  button clicks, hover states, game events, or ambient loops, and when
  they mention @joycostudio/suno, Suno, AudioSource, Voice, or Mixer.
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# JOYCO Suno — Sound Effects Skill

This skill installs and scaffolds sound-effect playback in a web app using `@joycostudio/suno`. It picks the right entry point (vanilla vs React, with or without a Mixer), writes a typed manifest, and wires an unlock gesture so audio actually plays.

---

## When to use

Use this skill when the user wants to:
- Play a sound on button click, hover, or any UI interaction.
- Add ambient loops or game audio.
- Overlap or crossfade multiple sounds.
- Apply effects (filter, reverb, delay) to sounds.
- Slow down / speed up all audio globally (pause menus, bullet time).

Do **not** use this skill for:
- Building a music player UI with transport controls — this library is optimized for sound effects, not media playback UIs.
- Audio recording, streaming, or microphone input.

---

## Mental model

The library has four layers. Keep this picture in mind when scaffolding:

```
Voice → AudioSource bus → [optional effect chain] → masterOutput → speakers
```

- **`Suno`** — one per app. Holds a registry of named assets (the manifest) and the underlying `WebAudioPlayer`.
- **`AudioSource`** — one per loaded asset. Owns the decoded `AudioBuffer` and a per-source `GainNode` bus.
- **`Voice`** — one per `play()` call. Owns its own buffer source + gain node, so multiple voices of the same asset play independently and can be controlled in isolation.
- **`Mixer`** (optional) — sits on top of `Suno` and adds fade-in / fade-out on a requestAnimationFrame loop.

Every `source.play()` spawns a **new Voice** — overlapping playback is free. Voices auto-dispose on `ended` or `stop`.

---

## Install

```bash
pnpm add @joycostudio/suno
# or: npm i @joycostudio/suno   /   yarn add @joycostudio/suno   /   bun add @joycostudio/suno
```

React users import from the `/react` subpath — same package, no separate install:

```ts
import { SunoProvider, useSuno, useUnlock } from '@joycostudio/suno/react'
```

---

## Decide the entry point

Ask these questions in order:

1. **React app?** → Use `SunoProvider` + hooks. Otherwise → `new Suno({ manifest })` directly.
2. **Does the UX need fades (ambient crossfade, music layers, fade-in intros)?** → Wrap Suno with `Mixer`. Otherwise → Suno alone is enough.
3. **Typed keys wanted?** → Declare the manifest with `as const` and pass `typeof MANIFEST` to `Suno<M>` / `useSuno<M>()`.

---

## React setup (canonical)

Define the manifest once, mount the provider at the root of the audio-using subtree, gate audio behind an unlock gesture.

```tsx
// lib/audio/manifest.ts
export const AUDIO_MANIFEST = {
  click:    { src: '/audio/click.ogg' },
  hover:    { src: '/audio/hover.ogg', volume: 0.4 },
  ambient:  { src: '/audio/ambient.ogg', loop: true, volume: 0.6 },
} as const

export type AudioManifest = typeof AUDIO_MANIFEST
```

```tsx
// app/providers.tsx (or wherever you mount providers)
'use client'
import { SunoProvider } from '@joycostudio/suno/react'
import { AUDIO_MANIFEST } from '@/lib/audio/manifest'

export function AudioProvider({ children }: { children: React.ReactNode }) {
  return <SunoProvider manifest={AUDIO_MANIFEST}>{children}</SunoProvider>
}
```

```tsx
// components/sfx-button.tsx
'use client'
import { useSuno, useUnlock } from '@joycostudio/suno/react'
import type { AudioManifest } from '@/lib/audio/manifest'

export function PlayClick() {
  const suno = useSuno<AudioManifest>()
  const { unlock, unlocked } = useUnlock()

  const handleClick = async () => {
    if (!unlocked) await unlock()
    suno.get('click').play()
  }

  return <button onClick={handleClick}>Click me</button>
}
```

The first interaction both unlocks the audio context *and* plays the sound — one gesture covers both.

---

## Vanilla setup

```ts
import { Suno } from '@joycostudio/suno'

const suno = new Suno({
  manifest: {
    click:   { src: '/audio/click.ogg' },
    ambient: { src: '/audio/ambient.ogg', loop: true },
  },
})

document.querySelector('#start')!.addEventListener('click', async () => {
  await suno.unlock()
  await suno.loadAll()
  suno.get('ambient').play()
})
```

`unlock()` must be called inside a user-gesture handler (click, keydown, touch) — the browser blocks audio until then.

---

## Recipes

### One-shot SFX (rapid, overlapping)

```ts
suno.get('click').play() // each call spawns a new Voice — clicks layer cleanly
```

### Looping ambient with manual stop

```ts
const voice = suno.get('ambient').play()
// later:
voice.stop()
```

### Exclusive playback (cut in-flight voices)

Useful for voice-over lines or any "one at a time" source:

```ts
suno.get('vo-line-1').play({ exclusive: true })
```

### Crossfade with Mixer

```ts
import { Mixer } from '@joycostudio/suno'

const mixer = new Mixer({ suno, fadeOutDuration: 1, fadeInDuration: 1 })

mixer.stopByKey('ambient-day',   { fadeOut: 2 })
mixer.play('ambient-night',      { fadeIn: 2, loop: true })
```

In React, create the Mixer in a ref so it persists across renders:

```tsx
const mixerRef = useRef<Mixer | null>(null)
if (!mixerRef.current) mixerRef.current = new Mixer({ suno })
useEffect(() => () => mixerRef.current?.dispose(), [])
```

### Effect chain (filter, reverb, etc.)

Build the chain once, reuse the head node across many plays:

```ts
const ctx = suno.player.audioContext
const filter = ctx.createBiquadFilter()
filter.type = 'lowpass'
filter.frequency.value = 800

const fx = suno.effect(filter)  // wires filter → masterOutput, returns filter
suno.get('click').play({ output: fx })
suno.get('hover').play({ output: fx })
```

For reusable effect modules (classes that wrap a node graph, expose a `chain` head, and are safe to declare at module scope alongside `suno`), follow the pattern in [`writing-effects.md`](./writing-effects.md). It covers SSR-safe construction, `attach()` / `detach()` lifecycle, buffered params, parallel topologies (dry/wet), and per-voice routing.

### Global slowmo / fast-forward

Affects every live voice and seeds new ones. Great for pause menus:

```ts
suno.setPlaybackRate(0.5)   // tape-style slowmo (half speed, octave down)
suno.setPlaybackRate(1)     // back to normal
```

### Volume levels (know which knob to turn)

- `voice.setVolume(v)` — one specific voice.
- `source.setVolume(v)` — the per-source bus (scales every voice on that source).
- `source.setDefaultVolume(v)` — initial volume for newly-spawned voices.
- `suno.setMasterVolume(v)` — global master gain.

They multiply: `masterVolume × sourceBusVolume × voiceVolume`.

---

## React hooks reference

- **`useSuno<M>()`** — the Suno instance, typed by your manifest. Throws if no provider.
- **`useUnlock()`** → `{ unlock, unlocked }`. Call `unlock` inside a user-gesture handler.
- **`useSource(key)`** → `{ source, isPlaying, voices, volume, loop, duration } | null`. Reactive.
- **`useVoice(voice)`** → `{ state, isPlaying, currentTime, duration, volume, loop, playbackRate, effectivePlaybackRate }`. Reactive snapshot of a single voice.
- **`usePlaying()`** → array of `{ key, definition, source, voice }` for every live voice.
- **`useSunoState()`** → `{ state, isPlaying, masterVolume, playbackRate, unlocked }`.

All hooks use `useSyncExternalStore` — SSR-safe, no hydration mismatches.

---

## Pitfalls

1. **Playing before unlock.** Browsers block audio until the user interacts. Always gate the first `play()` behind a click/keydown. In React, use `useUnlock` and check `unlocked` before playing — or call `unlock()` inside the same handler as the play.
2. **SSR.** `new Suno()` is safe on the server (the player lazy-inits). But touching `suno.player.audioContext` / `masterOutput` from server code throws. Put audio calls in client components or behind `useEffect`.
3. **Stale Voice references.** After `ended` or `stop`, the Voice is disposed. Methods become no-ops, but don't design flows that assume a Voice lives forever — subscribe to `voice.on('ended', ...)` or use `useVoice` for reactive state.
4. **Hot reload.** The `SunoProvider` disposes Suno on unmount. In dev, retaining voice refs across a remount will log "context closed" errors — recreate refs on the new provider.
5. **Leaking looping voices.** One-shot voices clean themselves up on `ended`. Looping voices stay alive until you call `.stop()` — keep the reference, or use `source.stopAll()` / `suno.stopAll()` / `mixer.stopByKey()`.

---

## Typical file layout

```
lib/audio/
  manifest.ts        # AUDIO_MANIFEST + types
  provider.tsx       # <AudioProvider> mounting SunoProvider
  use-sfx.ts         # optional hook wrapping common plays (useSfx().click())
app/
  layout.tsx         # <AudioProvider> wraps children
```

A thin `useSfx` wrapper keeps call sites short:

```tsx
// lib/audio/use-sfx.ts
'use client'
import { useSuno, useUnlock } from '@joycostudio/suno/react'
import type { AudioManifest } from './manifest'

export function useSfx() {
  const suno = useSuno<AudioManifest>()
  const { unlock, unlocked } = useUnlock()
  const play = async (key: keyof AudioManifest) => {
    if (!unlocked) await unlock()
    suno.get(key).play()
  }
  return { play }
}
```

Then at the call site:

```tsx
const { play } = useSfx()
<button onClick={() => play('click')}>OK</button>
```

---

## Quick self-check before finishing

- [ ] Manifest declared with `as const` and an exported type.
- [ ] `SunoProvider` / `new Suno()` instantiated exactly once at the appropriate scope.
- [ ] An unlock gesture wired to a real user interaction (not a `useEffect` on mount).
- [ ] Looping voices have a clear stop path.
- [ ] Effect chains (if any) built once, not per-play.
- [ ] No server-side reads of `suno.player.audioContext`.
