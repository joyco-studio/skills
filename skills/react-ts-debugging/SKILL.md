---
name: react-ts-debugging
description: >-
  Debug React and TypeScript bugs through a collaborative, log-driven loop with
  the developer instead of guessing from source code. Use this whenever a user
  is hunting a bug in a React/TS/JS app — "why is my component re-rendering",
  "this state won't update", "useEffect runs twice / infinitely", "my fetch
  returns undefined", "this works sometimes but not always", "the value is wrong
  but I don't know why", or any "help me figure out what's going on" request.
  Trigger it even when the user just pastes a broken component and asks what's
  wrong: rather than eyeballing the file, run a real debugging session where you
  place strategic console logs, the developer runs the app and collects the
  output, and that output drives the next step. Prefer this over read-and-guess
  debugging.
---

# React/TS Debugging — a log-driven session with the developer

## The one idea that matters

You cannot see the running app. The developer can. So treat every bug as a
**joint investigation** where your job is to place instruments (logs) that turn
the invisible runtime into evidence, and the developer's job is to run the app,
reproduce the bug, and bring the evidence back. The logs they collect ARE the
data you reason over — not the source code.

Reading a file and announcing "ah, the bug is on line 42" is a guess. Sometimes
the guess is right, but you have no way to know, and when it's wrong you've
spent the user's trust and possibly broken working code. A log that fires (or
fails to fire) with a specific value is a *fact*. Build the diagnosis out of
facts.

This reframes the whole interaction: you are not a code reader who occasionally
asks questions. You are running an experiment, and the developer is your hands
and eyes in the lab.

## The loop

Run this cycle. Do not skip ahead to a fix.

1. **Reproduce** — establish the exact steps that trigger the bug and what the
   developer expects to see vs. what actually happens. No repro, no debugging.
2. **Hypothesize** — state 1–3 specific, falsifiable guesses about the cause.
   Each one should predict a *different* log output, so the logs can tell them
   apart.
3. **Instrument** — add a focused set of logs designed to confirm or kill those
   hypotheses. Few, targeted, well-labeled. Then **STOP**.
4. **Hand off** — tell the developer exactly what to do (which steps to perform)
   and ask them to paste back the full console output.
5. **Read the evidence** — interpret what came back, *including logs that should
   have fired but didn't*. Update or narrow the hypotheses.
6. **Iterate** — if the cause isn't pinned down yet, go back to step 2 with a
   sharper question. If it is, go to 7.
7. **Confirm, fix, clean up** — only fix once a log has *demonstrated* the root
   cause. Then remove every debug log you added.

The discipline that makes this work is the **STOP after step 3**. Resist the
urge to add logs and immediately also rewrite the logic. If you change behavior
before you understand it, you contaminate the experiment and you'll never know
whether your "fix" fixed the real bug or just moved it.

## Step 1 — Reproduce before you instrument

Ask for, or confirm, three things:

- **Repro steps**: the precise sequence ("load the page, type in the search box,
  click Submit"). "It's broken" is not a repro.
- **Expected vs. actual**: what *should* happen and what *does*. The gap between
  these is what your logs will measure.
- **Consistency**: every time, or intermittent? Intermittent bugs (race
  conditions, stale closures, ordering) need logs with timestamps and counters;
  deterministic bugs don't.

If the developer can't reproduce it reliably yet, that's the first thing to
solve — sometimes a log that records inputs is how you *find* the repro.

## Step 2 — Hypothesize out loud

Before adding a single log, write down what you think might be happening and why.
Good hypotheses are specific and competing:

> Either (a) the effect never runs because its dependency array is wrong, or
> (b) it runs but `fetchUser` returns before state updates, or (c) it runs and
> sets state but a second render overwrites it. A log at effect entry + one
> before/after setState will tell us which.

Stating hypotheses keeps the log set small and purposeful. If you can't say what
a log is testing, don't add it — log spam buries the signal.

## Step 3 — Designing logs that actually teach you something

### Use one greppable marker on every line you add

So that both you and the developer can find and remove all instrumentation in
one pass, tag every inserted line with a consistent token. Use a `// @debug`
comment on the code line and a recognizable prefix in the log string:

```ts
console.log("[DBG] Cart.render", { itemCount: items.length }); // @debug
```

At cleanup time, `grep -rn "@debug" src/` finds every line. Pick the token at
the start of the session and tell the developer what it is, so they trust that
nothing permanent is being smuggled into their codebase.

### Make logs say where they are and carry context

A lone `console.log(value)` in a 200-component app is useless. Always include:

- **A location label**: `"[DBG] useAuth effect"` or `"[DBG] Row.onClick id=42"`.
- **The relevant data, as a snapshot of just the fields that matter** — not the
  whole giant object. Logging a live object reference can mislead, because the
  browser console may show its *later* mutated state; snapshot primitives or
  `structuredClone` the slice you care about.
- **For intermittent/ordering bugs**, a counter and/or time: `console.count()`
  for render counts, or include `performance.now()`.

### Pick the right console method for the question

- `console.log({ a, b })` — values at a point. Wrap in an object so labels stick.
- `console.count("[DBG] Foo render")` — how many times something runs (re-renders,
  effect fires). Perfect for "why does this run N times".
- `console.table(arrayOfObjects)` — comparing rows of data.
- `console.trace("[DBG] who called setOpen")` — the call stack that led here, for
  "what is triggering this".
- `console.group` / `console.groupEnd` — bracket a render or a handler so related
  logs read as one unit.
- `console.warn` / `console.error` — make a specific "this branch should never
  run" marker stand out in a noisy console.

### Log the boundaries, not the middle

Bugs almost always live at a boundary. Instrument the edges and let the values
tell you which side is wrong:

- **Render boundary** — log props in and derived values out, plus a render count.
- **State transitions** — log immediately before `setState` (the value you're
  setting) and in the next render (the value you actually got). React batches and
  defers, so "I set it to X" and "it became X" are different events.
- **Effect boundaries** — log on effect entry, log the dependency values, and log
  in the cleanup function. This catches wrong deps, double-fires, and missing
  cleanup.
- **Async boundaries** — log right before an `await`/`fetch`, right after it
  resolves, and in the `catch`. The most common React bug shape is "state set
  after the component moved on."
- **Event handlers** — log on entry with the event's relevant fields.
- **Branches** — drop a labeled log in each side of a suspicious `if`/ternary so
  you can see which path executed.

### Reveal runtime reality that TypeScript hides

TypeScript types are erased at runtime. A value typed `User` can be `undefined`,
a `number` can arrive as a `"42"` string from an API, and an `any` or a `!`
assertion can hide a lie. When the bug smells like a type/shape mismatch, log the
*actual* runtime nature, which the types cannot guarantee:

```ts
console.log("[DBG] api payload", {
  value: data,
  type: typeof data,
  isArray: Array.isArray(data),
  ctor: data?.constructor?.name,
}); // @debug
```

See `references/log-recipes.md` for a catalog of ready-to-paste instrumentation
for the common React/TS bug shapes (re-render storms, stale closures, infinite
effects, StrictMode double-invocation, lost state, race conditions, narrowing
failures, `===` identity surprises). Read it once you have a hypothesis about
*which shape* the bug is, and adapt the matching recipe.

## Step 4 — Hand off cleanly, then wait

When the logs are placed, end your turn with a clear, short handoff:

- Confirm what you added and where (and the `@debug` marker, so they know it's
  removable).
- Give the **exact actions** to perform to hit the bug.
- Ask them to **copy the entire console output** and paste it back — not a
  summary, not "it still doesn't work." The raw log is the data; their
  paraphrase isn't.
- If output could be huge, tell them to clear the console first, do the one
  action, then copy — so the relevant slice is isolated.

Then stop and let them run it. Do not pre-write the fix "to save a round trip."
You don't yet know what you're fixing.

## Step 5 — Read what comes back (and what didn't)

When the developer pastes logs, treat them as evidence:

- **Compare to predictions.** Which hypothesis does this output support, and
  which does it rule out? Say so explicitly.
- **Mind the absences.** A log you placed that *didn't* appear is often the
  biggest clue: the effect never ran, the branch was never taken, the handler
  never fired. "The dog that didn't bark" pins down the bug as precisely as a log
  that did.
- **Watch order and counts.** Three renders where you expected one, a cleanup
  that runs after the next effect, a resolve that lands after unmount — ordering
  is where React bugs hide.
- **Don't over-read one data point.** If the logs are ambiguous, that's a signal
  your instrumentation wasn't sharp enough; design a better single experiment
  rather than guessing between possibilities.

If the evidence doesn't yet name the cause, go back to step 2 with a narrower
question and a smaller, sharper log set. Each round should *halve* the search
space, like a binary search through the execution.

## Step 7 — Confirm, fix, and clean up

- **Confirm first.** Before fixing, you should be able to point at a specific log
  line and say "this value being X here is the bug." If you can't, you're still
  guessing — keep instrumenting.
- **Fix the cause, not the symptom.** Logs often reveal that the visible error is
  downstream of the real problem. Fix where the data first goes wrong.
- **Verify with the same logs.** Have the developer re-run the original repro with
  the instrumentation still in place to confirm the values are now correct, then
  remove the logs. Confirming the fix is part of the experiment.
- **Remove every debug line.** `grep -rn "@debug" src/` (or the chosen token) and
  delete them all. Leaving instrumentation behind is a bug of its own — noisy
  consoles, leaked data, confused teammates.

## Anti-patterns — don't do these

- **Read-and-guess.** Announcing the bug from the source without evidence. Even
  if you're confident, propose the log that would confirm it; confidence is not
  data.
- **Fixing while diagnosing.** Changing logic in the same turn you add logs. You
  lose the ability to attribute the change.
- **Log spam.** Twenty logs at once. You can't tell which matters, and neither
  can the developer. Few, labeled, hypothesis-driven.
- **Logging giant live objects.** They're hard to read and may display mutated
  state. Snapshot the fields that matter.
- **Accepting "it still doesn't work."** That's not evidence. Ask for the actual
  console output every time.
- **Forgetting cleanup.** Shipping `[DBG]` logs to production.

## When logs aren't the right instrument

The log loop is the default and the priority, but name the better tool when it
clearly fits, and still drive it collaboratively:

- **React DevTools** — for "why did this re-render" (the Profiler's "why" data)
  and inspecting current props/state without editing code.
- **Network tab** — when the question is what the server actually returned;
  pair it with a log of how the client parsed it.
- **Breakpoints / `debugger`** — when you need to inspect a rich object tree or
  step line-by-line; the developer pastes back what they see at the breakpoint,
  same loop.
- **A minimal reproduction** — when the app is too noisy to isolate the bug,
  guide the developer to strip it down. Often the act of minimizing reveals it.

Whatever the instrument, the contract is the same: you propose the experiment,
the developer runs it, the result drives the next step.
