# SEL-5 — `trigger: 'always'` + `copy: true` + a state-writing handler is an unbounded clipboard loop

P1. Found by the **blind re-audit**, 2026-09-07 — in code that had just been verified by the agent
that wrote it.

## Measured

```
+1009ms  copyEvents=4820
+2013ms  copyEvents=9551
+3016ms  copyEvents=14232
+4019ms  copyEvents=18637     ← linear, never terminates. rAF latency 4ms.
```
**~4,700 real clipboard writes per second, indefinitely.** The page stays visually responsive, so
it looks fine while it hammers the system clipboard.

## Why Vue's guard does not catch it

The cycle crosses a **promise boundary**:
`startCopy → writeText().then(settle → dispatch select-text-copy → handler writes state → render →
updated → 'always' → select → startCopy)`. Vue's recursive-update guard only sees synchronous
re-entry, so it never fires.

Isolation, same page, same shape:

| binding | outcome |
|---|---|
| `trigger:'always'` + `@select-text` handler, no copy | stops at **100** — Vue's guard catches it |
| `trigger:'always'` + `copy` + `@select-text-copy` handler | **unbounded** |
| `trigger:'always'` + `copy`, no handler | 1 write per render — no loop |
| `trigger:'edge'` (default) + `copy` + handler | **1 write total** — the edge is spent |

## Why it matters despite not being the default

It needs the explicit `always` + `copy` pairing — but **the README's own `select-text-copy` example
is `toast(…)`**, which is a state write. The documentation demonstrates the shape that loops, and
nothing rules the pairing out or warns at runtime.

## Fix — decide, do not guess

Options to weigh: refuse `copy` under `trigger: 'always'` with a `warnOnce`; suppress the copy when
the resolved text is unchanged since the last successful write (an idempotence guard, the same shape
`v-dropzone` added for its attribute writes); rate-limit or collapse in-flight writes per element;
or leave the behaviour and document the pairing loudly. **The first two are real fixes; the last is
not, given the README teaches the failing shape.**

## Also from the same re-audit

- The `copy: true` + non-click **warning fires even when `enabled: false`** — card 15 mounts
  disabled and still warns. Cosmetic.
- **`data-select-text-copy` is written unconditionally on every attempt**, with no idempotence
  guard. A consumer with a `MutationObserver` on the host has a second loop source. Not exercised
  end to end, but it is the exact mechanism that hung the `v-dropzone` tab.

## Acceptance

- A browser regression that reproduces the loop **before** the fix and terminates after — count the
  writes, do not eyeball it.
- The README's `select-text-copy` example either stops being a state write or the pairing is
  guarded.
- Do not regress what the re-audit confirmed: 30 checks across 15 cards, all six empty-host rows
  (zero events, decoy selection survives), the full activation table, `useSelectText`'s contract,
  and the keyboard affordance including the author's own `tabindex`/`role` being left alone.
