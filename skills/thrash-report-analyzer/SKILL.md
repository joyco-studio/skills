---
name: thrash-report-analyzer
description: >
  Analyze a bye-thrash layout thrashing report array. Parses stack traces,
  identifies user-code functions causing forced reflows, locates the offending
  style-write → layout-read pairs in source files, and produces a structured
  fix-suggestion report.
user-invocable: true
argument-hint: <report.json>
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# Layout Thrashing Report Analysis

Analyze a `bye-thrash` report and produce actionable fix suggestions for every layout thrashing instance found.

## Usage

```
/thrash-report-analyzer <path-to-report.json>
```

The argument is the absolute path to a JSON file containing the `bye-thrash` report array. If no file path is provided, ask the user to paste the report JSON directly.

## Input format

Each entry in the report array has at minimum:

- `prop` — the layout property that was read (e.g. `offsetWidth`, `clientHeight`)
- `count` — how many times this thrash was observed
- `stack` — a JavaScript call stack string captured at the moment of the layout read

## Workflow

Follow these steps in order for each entry in the report.

### Step 1 — Parse the stack trace

Split the `stack` string into individual frames. For each frame, extract:
- Function name (or `<anonymous>`)
- Source file path
- Line and column number

**Ignore frames from:**
- `bye-thrash` internals — any frame containing `Thrash.onLayoutRead`, `onRead`, or anonymous wrappers from the library's property patches
- React internals — `executeDispatch`, `processDispatchQueue`, `batchedUpdates`, `dispatchEvent`, `commitRoot`, `performSyncWorkOnRoot`, `renderRootSync`, `workLoop`, and similar scheduler/reconciler frames
- Next.js / webpack / turbopack internals — frames in `next/dist`, `webpack-internal`, `turbopack`, or `__next`

### Step 2 — Identify the user-code function

Walk the filtered frames top-to-bottom. The **first frame** whose source file is inside the project (not in `node_modules`, `.next`, or `dist`) is the user-code function responsible for the thrash.

Record:
- Function name
- Source file path
- Line number

### Step 3 — Search the codebase for the function

Use Grep to locate the function definition. Search `.tsx`, `.ts`, `.jsx`, `.js` files. Exclude `node_modules`, `.next`, and `dist` directories.

If the function name is generic (e.g. `onClick`, `handleResize`), use the source file path from the stack trace to narrow the search.

### Step 4 — Read the source and find the write → read pair

Read the source file around the identified line. Look for:

**Style writes** (any of these before the read):
- `el.style.*` assignments
- `classList.add()`, `classList.remove()`, `classList.toggle()`, `classList.replace()`
- `setAttribute('style', ...)` or `setAttribute('class', ...)`
- `className` assignment
- `innerHTML` or `textContent` assignment (these invalidate layout)
- `insertBefore`, `appendChild`, `removeChild`, `replaceChild`

**Layout reads** (the property from the report's `prop` field):
- Dimension reads: `offsetWidth`, `offsetHeight`, `offsetTop`, `offsetLeft`, `clientWidth`, `clientHeight`, `clientTop`, `clientLeft`
- Scroll reads: `scrollTop`, `scrollLeft`, `scrollWidth`, `scrollHeight`
- Computed style: `getComputedStyle()`, `currentStyle`
- Bounding rect: `getBoundingClientRect()`
- Other: `innerText`, `focus()`, `scrollTo()`, `scrollBy()`, `scrollIntoView()`

### Step 5 — Determine the fix

Choose the appropriate fix strategy:

| Scenario | Fix |
|----------|-----|
| Read is only used for comparison/logging | Move the read **before** the write |
| Read feeds into the write value | Cache the read before the write, or use `requestAnimationFrame` to defer the write |
| Multiple elements written then read in a loop | Batch all reads first, then batch all writes |
| Write is visual-only and read can be deferred | Wrap the read in `requestAnimationFrame` |
| Pattern is in a resize/scroll handler | Debounce or throttle the handler; use `ResizeObserver` instead |

### Step 6 — Generate the report

Output findings in this format for each entry:

```
### Entry: `{prop}` (count: {count})

- **Source file:** `path/to/file.tsx`
- **Function:** `functionName` (line X)
- **Write:** `el.style.fontSize = '24px'` (line Y)
- **Read:** `window.getComputedStyle(el).fontSize` (line Z)
- **Fix suggestion:** [brief actionable suggestion]
```

After all entries, add a **Summary** section:

```
## Summary

- **Total thrash instances:** N
- **Unique locations:** M
- **Most frequent:** `prop` in `functionName` (K occurrences)
- **Highest impact fix:** [which fix would eliminate the most thrash instances]
```

## Edge cases

- **Stack trace points to minified code** — If the source map is not available and function names are mangled, note this in the report and suggest the user provide source maps or an unminified build.
- **Multiple writes before a single read** — Report all writes that contribute to the invalidation, not just the closest one.
- **Thrash happens inside a third-party library** — If the first user-code frame is a call into a library that internally thrashes, note the library name and suggest the user file an issue or wrap the call in a `requestAnimationFrame`.
- **No matching source found** — If Grep cannot locate the function, report the stack trace as-is and suggest the user verify the source is available in the working directory.
