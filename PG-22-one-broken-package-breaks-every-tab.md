# PG-22 — a single broken package renders every tab empty

Found by the KB-2 agent, 2026-09-13, while another package was mid-edit.

`playground/src/libraries.ts` imports **every** library at module scope. So one sibling with a
compile error — in this case `v-teleport-to`'s `calculate-position.ts` importing a
`MEASURE_EPSILON` that did not exist yet — takes down the entire app: every tab rendered an empty
`#app`, and `smoke` reported only *"No library tabs rendered"*.

**The failure does not name the cause.** An agent working on `v-keyboard-navigation` lost part of a
run to a defect in `v-teleport-to`, and the only signal was a blank page.

This is the same shape as the other harness findings this month: the instrument fails in a way that
does not say why. See PG-14 (a compiler/runtime mismatch that silently disabled `updated`), PG-15
(a scoped alias silently falling back to source), PG-18 (`smoke` reporting 0 cards with no error),
PG-21 (`cdp.send` hanging forever with no timeout).

## What to fix

- **Isolate the import.** A tab whose library fails to load should render an error card naming the
  package and the error; every other tab should work. Dynamic `import()` per tab, or a try/catch
  around each registration, rather than one module-scope barrel.
- **`smoke` must say which package.** *"No library tabs rendered"* is the least useful possible
  message for the most common cause.
- The KB-2 agent added `PLAYGROUND_UNALIAS=<dir>` to `vite.config.ts` as a workaround — resolve a
  named package from `node_modules` instead of the sibling source for one run. Keep it, document
  it, or replace it with the isolation above; do not leave it as folklore.

## Why it matters beyond convenience

Multiple agents work in sibling packages concurrently here, and each one's source is aliased live
into the playground. Under that model an unrelated package's half-finished edit is not an edge case
— it is the normal condition, and it currently costs whole runs.
