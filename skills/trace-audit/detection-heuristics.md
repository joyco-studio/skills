# Detection Heuristics

Reference for all anomaly detection patterns used by the trace-audit skill. Each category lists the grep pattern, what fields to extract, and severity thresholds.

All durations in Chrome traces are in **microseconds** (µs). Convert to milliseconds for reporting: `ms = dur / 1000`.

---

## 1. Long Tasks

**What:** Main-thread tasks that block interactivity.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"RunTask"` |
| Duration field | `"dur":` (µs) |
| Threshold — Warning | dur > 50000 (>50ms) |
| Threshold — Critical | dur > 200000 (>200ms) |
| Extract | `ts`, `dur`, and the calling function from `args` if present |

**How to detect:** Grep for `RunTask`, then parse `dur` values from each matching line. Filter to dur > 50000.

---

## 2. Layout Thrashing

**What:** Rapid cycles of `InvalidateLayout` followed by `Layout` within the same animation frame, indicating JS is repeatedly reading then writing layout properties.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"InvalidateLayout"` and `"name":"Layout"` |
| Threshold — Warning | >10 InvalidateLayout→Layout pairs per second |
| Threshold — Critical | >30 pairs per second |
| Extract | `ts` of each event, group into 1-second windows |

**How to detect:** Grep for both event names. Pair them by proximity in timestamp. Count pairs per second.

---

## 3. Forced Reflows

**What:** `Layout` events triggered synchronously by JavaScript (have a `stackTrace` in their args).

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"Layout"` (then check for `stackTrace` in context) |
| Multiline grep | Pattern: `"name":"Layout"` with `stackTrace` nearby |
| Threshold — Warning | Any occurrence |
| Threshold — Critical | >5 occurrences or any with dur > 10000 |
| Extract | `ts`, `dur`, top of `stackTrace` (function name + line) |

**How to detect:** Grep for `"name":"Layout"` with context lines, then look for `stackTrace` in surrounding lines. Alternatively, use a multiline grep for `Layout.*stackTrace`.

---

## 4. rAF Ticker Loops

**What:** Excessive `requestAnimationFrame` calls indicating a busy animation loop or ticker.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"RequestAnimationFrame"` |
| Count over duration | total_rafs / trace_duration_seconds |
| Threshold — Warning | >120 rAF/sec sustained |
| Threshold — Critical | >240 rAF/sec sustained |
| Extract | Total count, rate per second |

**How to detect:** Count all `RequestAnimationFrame` events. Divide by trace duration in seconds. Also check for `FireAnimationFrame` and `CancelAnimationFrame` to understand the full picture.

---

## 5. Style Recalc Storms

**What:** Expensive `UpdateLayoutTree` (style recalculation) events, often caused by large DOM or complex selectors.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"UpdateLayoutTree"` |
| Duration field | `"dur":` (µs) |
| Element count field | `"elementCount":` in args |
| Threshold — Warning | dur > 5000 (>5ms) OR elementCount > 500 |
| Threshold — Critical | dur > 20000 (>20ms) OR elementCount > 2000 |
| Extract | `ts`, `dur`, `elementCount` |

---

## 6. Paint Storms

**What:** Expensive paint operations that block rendering.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"Paint"` |
| Duration field | `"dur":` (µs) |
| Threshold — Warning | dur > 3000 (>3ms) |
| Threshold — Critical | dur > 16000 (>16ms, misses a frame at 60fps) |
| Extract | `ts`, `dur`, clip rect dimensions from args |

---

## 7. GC Pressure

**What:** Garbage collection pauses that freeze the main thread.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"MajorGC"` or `"name":"V8.GC_MARK_COMPACTOR"` or `"name":"MinorGC"` |
| Duration field | `"dur":` (µs) |
| Threshold — Warning (Major) | dur > 10000 (>10ms) |
| Threshold — Critical | dur > 50000 (>50ms) |
| Threshold — Warning (Minor) | dur > 5000 (>5ms) |
| Extract | `ts`, `dur`, `usedHeapSizeBefore`, `usedHeapSizeAfter` from args |

---

## 8. Cumulative Layout Shift (CLS)

**What:** Visual instability from elements shifting position unexpectedly.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"LayoutShift"` |
| Score field | `"score":` in args or `"value":` |
| Threshold — Warning | Cumulative score > 0.1 |
| Threshold — Critical | Cumulative score > 0.25 |
| Extract | Individual shift scores, cumulative total, `had_recent_input` field |

**How to detect:** Sum all `score` values from `LayoutShift` events where `had_recent_input` is false. Report both individual shifts and cumulative score.

---

## 9. Interaction to Next Paint (INP)

**What:** Responsiveness metric measuring the delay between user interaction and visual feedback.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"EventTiming"` |
| Duration field | `"dur":` (µs) |
| Threshold — Warning | dur > 200000 (>200ms) |
| Threshold — Critical | dur > 500000 (>500ms) |
| Extract | `ts`, `dur`, event type from args |

**How to detect:** Find `EventTiming` events and extract their durations. The worst (highest duration) represents the INP value.

---

## 10. Network Errors

**What:** Failed network requests visible in the trace.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"ResourceReceiveResponse"` |
| Status field | `"statusCode":` in args |
| Threshold — Warning | statusCode >= 400 |
| Threshold — Critical | statusCode >= 500 |
| Extract | `statusCode`, request URL from `ResourceSendRequest` with matching request ID |

**How to detect:** Grep for `ResourceReceiveResponse` and filter for `statusCode` values >= 400. Cross-reference with `ResourceSendRequest` events to get the URL.

---

## 11. Redundant Fetches

**What:** The same resource URL being fetched multiple times, wasting bandwidth and potentially causing race conditions.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"ResourceSendRequest"` |
| URL field | `"url":` in args |
| Threshold — Warning | Same URL base (without query params) fetched >2 times |
| Threshold — Critical | Same exact URL fetched >3 times |
| Extract | URL, count of duplicates |

**How to detect:** Collect all `ResourceSendRequest` URLs. Strip query parameters. Group by base URL and flag any with count > 2.

---

## 12. Script Evaluation

**What:** Long-running script parsing and compilation blocking the main thread.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"EvaluateScript"` or `"name":"CompileScript"` |
| Duration field | `"dur":` (µs) |
| Threshold — Warning | dur > 50000 (>50ms) |
| Threshold — Critical | dur > 200000 (>200ms) |
| URL field | `"url":` in args |
| Extract | `ts`, `dur`, script URL |

---

## 13. Long Animation Frames (LoAF)

**What:** Animation frames that took too long to render, a newer Chrome metric.

| Field | Detail |
|-------|--------|
| Grep pattern | `"name":"LoAF"` or `"name":"LongAnimationFrame"` |
| Duration field | `"dur":` (µs) |
| Threshold — Warning | Any occurrence (these are already flagged by Chrome) |
| Threshold — Critical | dur > 100000 (>100ms) |
| Extract | `ts`, `dur`, blocking duration from args |

---

## General Notes

- When extracting `dur` values from grep output, the value will appear as `"dur":12345` in the JSON. Parse the numeric value.
- Timestamps (`ts`) are in microseconds from trace start. Convert to seconds for the report: `sec = ts / 1000000`.
- Some events use `tdur` (thread duration) in addition to `dur` (wall duration). Prefer `dur` for reporting.
- Events with `ph: "X"` are complete events with a duration. Events with `ph: "B"` and `ph: "E"` are begin/end pairs — compute duration as `E.ts - B.ts`.
- When counts are very high (>1000 matches), use Grep with `output_mode: "count"` first to get totals before pulling individual lines.
