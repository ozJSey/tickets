# WBC-5 — `vue-write-behind` has no strategic brief and no entry in the root backlog

Deferred out of the 0.1.1 data-loss fix (audit finding 14, the half that `WBC-3` does not
already cover). Nothing here is a defect in the published package; it is the paperwork that
makes the package findable and maintainable by the next session.

## What is missing

1. **`instructions/vue-write-behind.md` does not exist.** Every other v-\* package has a
   per-package strategic brief and `CLAUDE.md` names it as required reading before touching a
   package. The design reasoning currently lives only in `DESIGNS.md` → WBC-1 and in
   `tickets/WBC-1-write-behind-survey.md`. The brief should carry: the wedge (TanStack Query
   cannot express write-behind; TanStack DB has the opposite semantics), the scope fence (a
   state outbox, not an operation log), the invariants H1–H6, and the three-facts-three-homes
   structure that came out of the 0.1.1 fix (`ARCHITECTURE.md`).
2. **No entry in the root `TASKS.md` or `PROGRESS.md`.** `grep` finds the package in neither.
   It is published on npm (`@ozjsey/vue-write-behind`), so the status table in `TASKS.md` is
   wrong by omission, and the 0.1.1 fix run has no line in the root run log.

## Why it was deferred rather than done

Both are shared, append-heavy files at the repo root, and two other agents were editing
`v-dropzone` and `v-copy` in the same session — a concurrent write to `TASKS.md` /
`PROGRESS.md` would have clobbered one of them. The brief itself is a new file and could have
been written, but a brief that contradicts a backlog nobody updated is worse than none.

## Also still open (tracked in `WBC-3`, restated so this file stands alone)

- no `playground/src/demos/vue-write-behind/` tab, so the package is not in the shared
  playground at all — `tickets/_STANDARDS.md` item 4;
- `playground.html` is a CDP fixture, not a copy-pasteable example: an inline import map
  pointing at `./node_modules/vue/...` and `./dist/vueWriteBehind.min.js`, with a comment
  telling the reader to hand-edit it first. For a package whose stated distribution model is
  source-copying, the only worked example a stranger gets is the 15-line README snippet in its
  bare form — nobody ever sees `debounce`, `retry: false`, `keys` or the batch writer in
  context.

**Note for whoever builds the tab:** the flagship card is the one the whole product rests on —
a slow server that answers with a transformed value, typed into continuously, where the field
must never flicker to the server's answer. `scripts/browser-check.mjs` already proves it over
CDP (11 keystrokes, 2 PUTs, every sampled value a prefix of what was typed) and, since 0.1.1,
also proves that `discard()` mid-flight does not open a second concurrent request. Port both.
