# DOC-2 — no `PROGRESS.md` or `TASKS.md` entry exists for any of this session's work

`CLAUDE.md` requires it: *"Update package's own `PROGRESS.md` (if exists) AND root `PROGRESS.md`
(cross-package run log)."* Several packages keep their own — `v-teleport-to`, `v-dropzone`,
`v-trap-focus`, `v-scroll-into-view`, `v-select-text`.

**I instructed every agent not to touch those files**, to avoid concurrent append conflicts while
up to six agents ran at once. That was the right call for the conflicts and the wrong call for the
record: two days of work — five packages repaired, two packages created, a P0 in the playground's
own compiler, and roughly twenty tickets — exist in `BOARD.md`, `DESIGNS.md`, `tickets/` and the git
log, but **nowhere in the logs the repo's own convention says to keep.**

One agent flagged the conflict explicitly rather than silently obeying, which is how this got caught.

## What is owed

- A root `PROGRESS.md` entry for the session: the audit programme and why it started, the compiler
  mismatch and what it invalidated, the five package repairs, the two new packages, and the
  verification rule that came out of it.
- Per-package `PROGRESS.md` entries for `v-teleport-to`, `v-dropzone`, `v-select-text`,
  `v-scroll-into-view`, `v-copy`.
- `TASKS.md`'s status table is stale in several rows — versions, test counts, published state, and
  the playground demo count.

**Write it once, serially, when no agents are running.** That is the only safe way, and it is why
it was deferred rather than skipped.
