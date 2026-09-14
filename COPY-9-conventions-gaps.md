# COPY-9 — `v-copy`'s three convention gaps: no brief, `[data-copied]`, unenforced boundaries

P3. Quality-audit "ARCHITECTURE.md claims nothing enforces" + the CONVENTIONS items. Deferred from
1.1.1 because none of them is a defect a consumer can hit — they are the reasons the 1.1.0 defect
was possible to introduce unnoticed.

## 1. There is no `instructions/v-copy.md`

`CLAUDE.md` calls `instructions/<package>.md` "the source of truth for intent" and
`CONVENTIONS.md:56` requires a per-package strategic brief. `v-copy` has none — which is why the
config/controller conflation was recorded nowhere and survived a release. `v-fit-children` gained
one on 2026-08-09; use it as the template.

The brief must state, at minimum, what 1.1.1's ARCHITECTURE.md now records: **an object binding has
exactly one role, and the role is asked of the object (`isReactive && !isReadonly`), never inferred
from its keys.** Plus the wedge (aggregated copy + history, per `CLAUDE.md`) and the naming decision
pending in COPY-3.

## 2. `[data-copied]` diverges from `data-<directive>-state`

CONVENTIONS ("feels native" #5) asks for CSS state hooks as `data-<directive>-state`;
`v-fit-children` ships `data-fit-children-state`. `v-copy` ships a boolean `[data-copied]`.

It is not obviously wrong — a copy has one transient state, so a boolean attribute reads better in
CSS than `[data-copy-state="copied"]` — but nothing decides that, and each package currently picks
its own. Either write the exception into CONVENTIONS with the reasoning, or rename at the next
major. **Do not rename in a patch**: `[data-copied]` is in the README, in five demos and in
consumers' CSS.

## 3. Three ARCHITECTURE.md invariants that nothing verifies

- *"`execute.ts` is the only place a copy happens"* — true by convention only. `clipboard.ts`
  exports `runCopy` to the whole package, and nothing stops the next feature from calling it and
  skipping both refusals.
- *"dependencies point strictly downward — no cycles"* — held at the 1.1.1 audit (re-checked after
  `history.ts` gained an import of `controller.ts`), but nothing verifies it. `state.ts` already
  imports types back out of `resolve.ts`; the first non-type import in that direction closes a cycle
  silently.
- The entry/`src/index.ts` export-parity hole **is fixed** (1.1.1 made `vCopy.ts` a wildcard
  re-export and added `npm run typecheck`) — no work left there.

A ~20-line `node --test` or vitest check over the import graph would cover the first two, and it is
portfolio-wide value, not v-copy value: every package makes the same claim in its ARCHITECTURE.md.

## Acceptance

- `instructions/v-copy.md` exists and names the ownership invariant.
- The `data-*` question is either written into CONVENTIONS as a deliberate exception or scheduled
  for the next major with a migration note.
- A module-graph test fails when `clipboard.ts`'s `runCopy` gains a second caller, or when a cycle
  is introduced.
