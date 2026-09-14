# PG-22 — one package that does not compile takes the whole playground down, silently

Opened 2026-09-13 from the `v-observe` audit close-out, which lost about half an hour to it.

## What happened

The playground would not boot at `HEAD`. Every tab, every library, blank page:

```
SyntaxError: The requested module '/@fs/…/v-teleport-to/src/constants.ts'
does not provide an export named 'MEASURE_EPSILON'
```

`v-teleport-to/src/calculate-position.ts:25` imports `MEASURE_EPSILON` from `./constants`, and
`constants.ts` never exported it — left behind by the same 13-agent wave that produced the WIP
checkpoint `3802fa6`. `v-teleport-to`'s own git status was clean, so this was committed, not
someone's uncommitted work in flight.

`main.ts` installs every library's directive at boot, so one unresolvable import is fatal for all
ten tabs. Nothing distinguished it from "the dev server is not up yet": `page.pageErrors` had the
message, but the runner's boot guard only reports *"the playground did not boot —
`window.__PLAYGROUND_VERSIONS__` is unset"*, which reads exactly like a slow start.

## What was done

`MEASURE_EPSILON = 0.5` was added to `v-teleport-to/src/constants.ts` with the doc comment the
call site implies (sub-pixel tolerance for "is the content taller than the clamp?", the value that
keeps a 39.531px host under a 40px clamp from reading as truncated and livelocking the flip).
`v-teleport-to`'s 806 tests pass with it. **That was an unblock by an agent working on another
package, not a review** — the `v-teleport-to` owner should confirm 0.5 is the intended value, and
whether the constant was meant to carry more than that one call site.

## What is actually worth fixing

**1. The boot guard should say what it saw.** `scripts/interactions.mjs` and `scripts/smoke.mjs`
both know the page errors by the time they give up. Printing the first one turns half an hour into
ten seconds:

```
The playground did not boot — window.__PLAYGROUND_VERSIONS__ is unset.
First page error: SyntaxError: The requested module '…/constants.ts' does not
provide an export named 'MEASURE_EPSILON'
```

**2. A package that does not typecheck should not be able to break every tab.** `pnpm typecheck`
covers the app and the demo SFCs; it does not compile each library's `src/` on its own. A missing
export inside a library is invisible until Vite resolves it at runtime. Cheapest fix: a
`pnpm typecheck:libs` that runs `tsc --noEmit` per aliased package, in CI and in the standing
checklist. It would have caught this before any browser was launched.

**3. Consider isolating `installLibraries`.** Ten `app.directive()` registrations behind one
try/catch means the tenth library's typo blanks the first nine. Registering each inside its own
try/catch, and rendering a per-tab "this library failed to load: <message>" banner, would keep the
other nine usable and make the failure legible. Weigh against `main.ts`'s deliberate
fail-loud-and-fatal design (PG-14 / PG-15) — the argument there was that a quiet degradation is
worse. A visible per-tab banner is not a quiet degradation.

## Acceptance

- Boot failures in `smoke.mjs` and `interactions.mjs` print the first page error.
- Either the per-library typecheck exists, or a note in `playground/README.md` says why not.
- Negative-control whichever you build: delete an export a library imports, confirm the new
  diagnostic names it, restore.
