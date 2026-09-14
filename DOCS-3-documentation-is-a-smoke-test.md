# DOCS-3 — the documentation view is a smoke test

Owner, 2026-09-13: *"We can treat documentation as smoke test too."*

`Documentation.vue` shipped mid-session and is wired into `App.vue`, rendering each package's own
README. **Nothing verifies any of it.** `scripts-status.mjs` reports `docs view checked 0/12`.

This is not a docs chore. The single most expensive class of defect this month was documentation
asserting behaviour the code does not have:

- `v-select-text`'s README recommended a `navigator.clipboard.writeText` call that **rejects in
  Chrome** on the package's own default trigger, surfacing as an uncaught `NotAllowedError`.
- Two `v-dropzone` README recipes were copy-paste broken — one sent `Bearer undefined`, the other
  threw `_ctx.fetch is not a function`.
- `v-fit-children`'s brief documented three exports that **do not exist** in the source, the dist or
  the published tarball.
- `vue-write-behind`'s README and ARCHITECTURE both promised, as an absolute guarantee, the exact
  thing that was broken.
- `v-teleport-to`'s `resolve-placement.ts` claimed a bad measurement "self-corrects when the
  animation settles". Nothing re-measures the host.

Every one was found by a human or an auditor reading carefully. None was found by a gate.

## What to check

1. **It renders, for all 12.** The view resolves each package by folder name; a package whose README
   is missing, unreadable, or renders empty is a failure, not a blank tab.
2. **Links resolve.** Every link in every rendered README, including the `ozjsey.github.io/#<id>`
   links now required by `_STANDARDS.md`, and cross-package "part of a set" links. A 404 in the docs
   of a published package is public.
3. **Code samples run.** This is the expensive one and the one that pays. Extract fenced code blocks
   from each README, compile them the way the playground compiles a demo, and fail on a block that
   does not compile. **Do not hand-copy the samples into the test** — extract them from the file, so
   the check cannot drift from the doc. The `v-dropzone` fix did exactly this and found both broken
   recipes.
4. **Claims are demonstrated.** Where a README states a behaviour, there should be a card or a check
   that shows it. A claim with neither is a reportable gap — the same shape as `BIG-1`'s "with and
   without" requirement, generalised.
5. **The published README is the one being checked.** For published packages the README is the npm
   page. If `files` excludes it, or the tarball's copy differs from the repo's, that is a finding.

## Scope note

Item 3 is the whole value and also the most work. If it has to be staged, do **extraction and
compilation** first — that alone would have caught the two `v-dropzone` recipes and the
`v-select-text` clipboard example. Items 1 and 2 are cheap; do them in the same pass.

## Acceptance

- A `docs` phase or spec that runs with the rest of the harness, not separately.
- `scripts-status.mjs`'s `docs view checked` column moves off 0.
- The check fails when a README claims something false — **prove it with a negative control**, by
  reintroducing one of the five defects above and confirming the gate goes red.
- Findings reported per package, not aggregated into a pass/fail.
