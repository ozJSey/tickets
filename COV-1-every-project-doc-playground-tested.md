# COV-1 — every project: playground + documentation + real testing

**Owner, 2026-09-05:**
> "I wanna see all projects having a doc and a playground. And heavily tested, people use
> libraries to use confirmed code, not for not working stuff."
> "Of course, we have a high standard we should maintain, what did you expect?"

Scope is **all ten existing projects plus the two new ones**, not just the new ones.
Depends on `tickets/DOCS-1-pages-deploy.md` for the docs mechanism and the Pages deploy.

## Measured state, 2026-09-05

| Package | Playground | Docs | Browser spec | Unit tests | Test kind |
|---|---|---|---|---|---|
| v-teleport-to | 12 cards | no | **no** | 337 | vitest |
| v-dropzone | 12 cards | no | yes | 283 | vitest |
| v-observe | 17 cards | no | **no** | 189 | vitest |
| v-select-text | 12 cards | no | yes | 176 | vitest |
| v-scroll-into-view | 9 cards | no | **no** | 109 | vitest |
| v-copy | 12 cards | no | **no** | 47 | vitest |
| v-fit-children | 9 cards | no | **no** | 43 | vitest |
| bigdecimal-string | **none** | no | **no** | 122 | vitest (`tests/*.spec.ts`) |
| dependency-grouper | **none** | no | **no** | **0** | 657-line hand-rolled node script |
| vue-provide-seeker | **none** | no | **no** | **0** | `vscode-test`, file has **0 cases** |

## The four gaps

**G1 — three projects have no demo surface.** Already P0 in `TASKS.md`. None is a Vue
directive, so the one-tab-per-directive shape does not fit as-is.
- `bigdecimal-string` — the clear win, and `TASKS.md` says do it first: a REPL card with two
  operands and an operation picker, showing the exact decimal string beside the IEEE-754 float
  answer. The value proposition becomes self-evident.
- `dependency-grouper` — a CLI. Needs a before/after `package.json` diff viewer driven by a
  canned fixture, since the real thing runs in a terminal against a repo.
- `vue-provide-seeker` — a VS Code extension; cannot run in a browser tab. The honest surface
  is a documentation card with a recorded GIF and a marketplace link. **Say that it is not
  interactive rather than faking a demo.**

**G2 — no project has a documentation view.** All ten. See `DOCS-1`; docs render from each
package README so they cannot drift from what ships on npm.

**G3 — five of seven `v-*` libraries have no browser verification.** 1,306 unit tests, all
jsdom, which has no layout, no real clipboard, no focus order, no `webkitGetAsEntry`. This is
the gap that let `v-teleport-to`'s `flip` sit dead in plain sight while every gate was green.
Missing specs: `v-teleport-to`, `v-observe`, `v-scroll-into-view`, `v-copy`, `v-fit-children`.
Coordinate with PG-11 — if the Playwright migration lands, write these as Playwright specs
rather than porting them twice.

**G4 — two published-or-shipped projects have no real tests.**
- `dependency-grouper` is **live on npm, 13 versions, and gets real downloads** with zero
  automated tests. It is also, per `CLAUDE.md`, the package whose file structure is admired.
  This is the sharpest violation of the owner's own standard in the repo.
  `TASKS.md` marks it "DO NOT TOUCH (but see P2)" — that guard was about not breaking a live
  package, not about leaving it untested. Raise with the owner before changing it.
- `vue-provide-seeker` ships a test file containing **zero test cases**. An empty test file is
  worse than no test file: from the outside it reads as coverage. Either populate it or delete
  it — do not leave it.

## Acceptance

- All 12 projects (10 existing + 2 new) have a playground surface and a documentation view,
  or an explicit, written statement of why an interactive demo is impossible
  (`vue-provide-seeker` is the only expected case).
- All 7 `v-*` libraries have browser specs; every demo card is either proven or listed as
  UNPROVEN with a reason. No card is silently uncounted.
- `dependency-grouper` has a real automated suite, or a recorded owner decision not to.
- `vue-provide-seeker`'s empty test file is populated or removed.
- `TASKS.md`'s P0 playground-coverage-gap item is closed or updated to match reality.

## Sequencing note

This is a large ticket and should be dispatched as one project per agent, not as a batch.
Recommended order: `bigdecimal-string` demo (highest value, self-contained) → the five missing
browser specs (highest bug-finding rate; two prior specs each found real library bugs) →
`dependency-grouper` tests → the docs views once `DOCS-1` settles the mechanism →
`vue-provide-seeker` last, since it is the least interactive.
