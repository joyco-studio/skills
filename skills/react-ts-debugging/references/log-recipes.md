# Log recipes for common React/TS bug shapes

Read this once you have a hypothesis about *which shape* the bug is. Find the
matching section, adapt the snippet, keep the `// @debug` marker so cleanup is
one `grep`. Each recipe says **what to suspect**, **what to log**, and **how to
read the result**.

## Contents

1. Re-render storm ("why does this render so many times")
2. Infinite or double `useEffect`
3. State that "won't update"
4. Stale closure (handler/effect sees old state)
5. Async race ("set state after unmount" / wrong value wins)
6. `useEffect` cleanup not running / running at the wrong time
7. Referential identity surprises (`useMemo`/`useCallback`/deps)
8. Props arriving wrong at a boundary
9. TypeScript lies: runtime shape ≠ declared type
10. Conditional render / branch never taken
11. Context value not updating consumers

---

## 1. Re-render storm

**Suspect:** new object/array/function identity each render, parent re-rendering,
or state set in render.

**Log:**
```ts
function Cart({ items }: { items: Item[] }) {
  console.count("[DBG] Cart render");               // @debug
  console.log("[DBG] Cart props identity", {        // @debug
    itemsRef: items,                                 // watch if this changes each time
    len: items.length,
  });                                                // @debug
  ...
}
```

**Read:** If the count climbs with no user action, something upstream is
re-rendering. If `itemsRef` is a brand-new array every render even when contents
are equal, the parent is recreating it — memoize at the source, not the child.

---

## 2. Infinite or double effect

**Suspect:** a dependency that changes every render (object/array/function), a
`setState` inside the effect feeding its own deps, or React 18 StrictMode
double-invoking in dev.

**Log:**
```ts
useEffect(() => {
  console.count("[DBG] sync effect fires");          // @debug
  console.log("[DBG] sync effect deps", { userId, filtersRef: filters }); // @debug
  return () => console.log("[DBG] sync effect cleanup"); // @debug
}, [userId, filters]);
```

**Read:** Fires exactly twice on mount with identical deps and nothing else
looping → StrictMode dev double-invoke, usually harmless. Fires forever → a dep
identity changes each render (look at `filtersRef`) or the effect sets state that
is (directly or transitively) in its own dep array.

---

## 3. State "won't update"

**Suspect:** setting then reading in the same tick, setting a derived value that a
later render overwrites, or mutating state in place instead of replacing it.

**Log:**
```ts
const [count, setCount] = useState(0);
console.log("[DBG] render sees count", count);       // @debug

function inc() {
  console.log("[DBG] before set", count);            // @debug
  setCount(count + 1);
  console.log("[DBG] after set (still old!)", count); // @debug — expected: unchanged
}
```

**Read:** "after set" showing the old value is **correct** React behavior, not the
bug — state updates are async and the closure is fixed for this render. If the
*next* render's "render sees count" also doesn't change, then the update truly
didn't take: check for in-place mutation (`state.push(x); setState(state)` keeps
the same reference and is skipped) or a later effect resetting it.

---

## 4. Stale closure

**Suspect:** an effect/handler/`setInterval` captured an old value because its
dep array told React not to refresh it.

**Log:**
```ts
useEffect(() => {
  const id = setInterval(() => {
    console.log("[DBG] tick sees count", count);     // @debug — frozen at mount value?
  }, 1000);
  return () => clearInterval(id);
}, []); // empty deps → closure frozen
```

**Read:** If "tick sees count" is stuck at the initial value while the UI shows a
higher number, the closure is stale. Confirms the fix direction: functional
updater (`setX(prev => ...)`) or correct deps.

---

## 5. Async race

**Suspect:** a fetch resolves after the component unmounted or after a newer
request, so the wrong/old response wins.

**Log:**
```ts
useEffect(() => {
  const reqId = ++requestSeq;                         // @debug module-level counter
  console.log("[DBG] fetch start", { reqId, query }); // @debug
  let alive = true;
  fetchResults(query).then((data) => {
    console.log("[DBG] fetch resolve", {              // @debug
      reqId, alive, n: data.length,
    });
    if (alive) setResults(data);
  });
  return () => { alive = false; console.log("[DBG] fetch abort", { reqId }); }; // @debug
}, [query]);
```

**Read:** If `reqId` 1 resolves *after* `reqId` 2, an older response can clobber a
newer one — that's the race. If a resolve logs `alive: false`, it's setting state
after cleanup. The `reqId` ordering in the console is the whole story.

---

## 6. Cleanup timing

**Suspect:** subscriptions not torn down, or cleanup running later/more often than
expected.

**Log:**
```ts
useEffect(() => {
  console.log("[DBG] subscribe", { channel });        // @debug
  const sub = subscribe(channel);
  return () => console.log("[DBG] unsubscribe", { channel }); // @debug
}, [channel]);
```

**Read:** Read subscribe/unsubscribe as a paired sequence. A subscribe with no
matching unsubscribe before the next subscribe = a leak. Order on `channel`
change should be: unsubscribe(old) → subscribe(new).

---

## 7. Referential identity surprises

**Suspect:** a `useMemo`/`useCallback`/dependency comparing by reference sees a new
object each render, so memoization never hits.

**Log:**
```ts
const config = useMemo(() => ({ sort, dir }), [sort, dir]);
console.log("[DBG] config identity", config);         // @debug — same ref across renders?
useEffect(() => {
  console.count("[DBG] effect that depends on config"); // @debug
}, [config]);
```

**Read:** If the effect count rises while `sort`/`dir` are unchanged, the memo deps
are themselves unstable, or something recreates `config`. React compares deps with
`Object.is` — two equal-looking objects are different references.

---

## 8. Props wrong at a boundary

**Suspect:** the bug is in the parent, not the child everyone's staring at.

**Log:** at the **top** of the child component body:
```ts
console.log("[DBG] <UserCard> received", { userId, name, onSave: typeof onSave }); // @debug
```

**Read:** If `userId` is already `undefined` here, stop debugging the child — the
parent passed garbage. Walk one level up and log there. This single log saves
whole sessions spent in the wrong file.

---

## 9. TypeScript lies (runtime shape ≠ type)

**Suspect:** an `any`, a `!` assertion, an `as` cast, or an untyped API response
let a wrong-shaped value through that the compiler swears is fine.

**Log:**
```ts
console.log("[DBG] payload shape", {                  // @debug
  value: data,
  type: typeof data,
  isArray: Array.isArray(data),
  keys: data && typeof data === "object" ? Object.keys(data) : null,
  ctor: data?.constructor?.name,
});
```

**Read:** A `number` field that logs `type: "string"` (common from JSON/query
params), a "User" that's actually `undefined`, an array that's `{}` — these are
invisible to TS and obvious in the log. Fix by validating/parsing at the boundary
(e.g., a schema check) rather than trusting the declared type.

---

## 10. Branch never taken

**Suspect:** a condition you assume is true isn't, so a render or handler silently
no-ops.

**Log:**
```ts
if (isReady) {
  console.log("[DBG] branch: isReady TRUE", { isReady, user }); // @debug
  return <Dashboard user={user} />;
}
console.warn("[DBG] branch: isReady FALSE — rendering null", { isReady }); // @debug
return null;
```

**Read:** Use the **absence** of the TRUE log as the clue — if you only ever see
FALSE, the gate condition is wrong, not the thing behind the gate. `console.warn`
makes the unexpected path pop visually.

---

## 11. Context not updating consumers

**Suspect:** consumers read a stale value because the provider value is a new
object each render (forcing re-renders) or never changes (so updates don't
propagate), or a consumer sits above the provider.

**Log:**
```ts
// in provider:
console.log("[DBG] Provider value identity", value); // @debug
// in consumer:
const ctx = useContext(MyContext);
console.log("[DBG] consumer reads ctx", ctx);         // @debug
```

**Read:** If the consumer's logged value never matches the provider's latest, the
consumer may be outside the provider tree, or reading a default. If the provider
value is a fresh object every render, every consumer re-renders — memoize it.
