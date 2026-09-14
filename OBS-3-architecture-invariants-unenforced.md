# OBS-3 — two `v-observe` architecture invariants are still enforced by nothing

Opened 2026-09-13 during the `v-observe` audit close-out (0.2.0). Low severity, cheap to close, and
worth closing because the same class of claim has already cost this package real bugs.

## Background

`v-observe/ARCHITECTURE.md` used to state five invariants and enforce none of them. Three were made
mechanical in 0.2.0 and the file now names the mechanism beside each rule:

- *"Each mode owns exactly one segment of `data-observe-state`"* — each segment lives inside that
  mode's own internal state and no mode holds a reference to another's, so `resize.ts` physically
  cannot write the mutate segment. (It previously could: segments were a shared mutable object
  written from six places across four files, `directive.ts` included.)
- *"The grammar and the string cannot drift"* — `observeStateAttribute()` is declared to return the
  exported `ObserveStateAttribute` template-literal type, and a test parses every string the
  directive writes during a full run.
- *"A gated mode's baseline is fully cleared on restore"* — `gate.ts` calls each mode's own
  `resetGateBaseline`, defined in the file that declares the fields, instead of hand-maintaining a
  list of somebody else's state.

## What is still only a comment

**1. Mode isolation.** *"`intersect.ts`, `resize.ts` and `mutate.ts` never import each other."* True
today; nothing checks it. The rule exists because the two places modes genuinely interact are
supposed to be explicit modules (`gate.ts`, `state-attribute.ts`) — an import between modes is how
that design quietly stops being true.

**2. No cycles.** *"Dependencies point strictly downward."* True today; nothing checks it. `tsup`
would happily bundle a cycle, and a cycle here would break the copy-paste story the whole `src/`
split exists for — a consumer taking `types + state + state-attribute + validate + <mode>` needs the
downward direction to hold.

## Suggested shape

A single test in `vObserve.test.ts` — no new dev dependency, no lint config:

- Read every file under `src/` with `node:fs`, pull the `from '...'` specifiers out with a regex,
  and build the import graph.
- Assert the three mode files never name each other.
- Assert the graph is acyclic (a DFS over ~12 nodes), and that `types.ts`, `state-attribute.ts` and
  `validate.ts` import nothing but each other and `types`.
- Read the module list out of `ARCHITECTURE.md`'s code fence and assert it matches the files on
  disk, so the map cannot rot either.

The trap to avoid: a test that reads the graph and asserts a snapshot of it. That passes forever and
means nothing. Assert the *property*.

## Acceptance

- The test exists and is negative-controlled: add `import { setupResize } from './resize'` to
  `mutate.ts`, confirm red, remove it. Do the same for a deliberate cycle. Say which in the commit.
- `ARCHITECTURE.md` updated so no rule is left claiming an enforcement it does not have.
