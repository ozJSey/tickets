# COPY-7 — `CopyResult.via` reports `'exec-command'` where no strategy ran

P2. Quality-audit finding 11. **Deferred from the 1.1.1 patch on purpose** — the honest fix widens
a published union, and widening a union a consumer *reads* is a compile-time break for anyone with
an exhaustive `switch (result.via)`. That is a minor, not a patch.

## What is wrong

`types.ts:17` documents `via` as "Which strategy ran" and `README.md:435` as "tells you which path
ran". Four paths stamp `via: 'exec-command'` without touching the clipboard:

- `execute.ts:47` — the disabled return.
- `execute.ts:94` — both refusals (`'empty'`, `'pending'`).
- `clipboard.ts:33` — the SSR branch (`unavailable: no DOM`).
- `controller.ts` — `ctrl.copy()` with no bound element (`error: 'no bound element'`).

The value is a filler constant that happens to typecheck. Telemetry built on it reports a wave of
legacy-`execCommand` copies on modern browsers — the exact signal someone would use to decide
whether the fallback can be dropped — for events where nothing was written.

1.1.1 documented the wart on the type ("only meaningful when a strategy *did* run — check
`success` / `error` first") rather than leaving the doc lying. That is a stopgap.

## The call

`export type CopyVia = 'clipboard-api' | 'exec-command' | 'none'`, and every path above reports
`'none'`. Ship in **1.2.0** with:

- a `CHANGELOG.md` **Changed** entry naming the compile-time impact on exhaustive switches;
- the README's "which path ran" line updated to list all three;
- a test per path asserting `via === 'none'`, and a positive control per path asserting a real copy
  still reports `'clipboard-api'` / `'exec-command'`.

Rejected alternative: making `via` optional (`via?: CopyVia`). It breaks every consumer that reads
`result.via` without a guard, which is worse than a widened union.
