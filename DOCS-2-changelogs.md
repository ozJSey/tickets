# DOCS-2 — changelogs, and the third view on the Pages site

Owner, 2026-09-06: *"We can publish gh-pages demo as well, but all changelogs needs to be adjusted
to show there. (once playgrounds also have documentations)"*

So the site is **three views per project — Playground · Documentation · Changelog** — and the
sequencing is explicit: **docs views land first, then Pages goes live.** DOCS-1 gates this; this
gates the deploy.

## Measured state, 2026-09-06 — worse than "needs adjusting"

| Package | CHANGELOG | Live on npm? |
|---|---|---|
| `v-copy` | **yes** (64 lines) | no |
| `v-dropzone` | **missing** | no |
| `v-fit-children` | **missing** | **YES — 11 versions** |
| `v-observe` | **missing** | no |
| `v-scroll-into-view` | **missing** | no |
| `v-select-text` | **missing** | no |
| `v-teleport-to` | **missing** | no |
| `vue-write-behind` | **missing** | no |
| `bigdecimal-string` | **missing** | **YES** |
| `dependency-grouper` | **missing** | **YES — 13 versions** |

**Nine of ten have no changelog, including all three already published.**

## The urgent part — version history was made today and never recorded

- **`v-teleport-to` is at 3.0.0.** It went 1.0.0 → 2.0.0 (`flip` became fit-based, a semantics
  change) → 3.0.0 (hide-when-reference-hidden became the default, and `overflow: 'hide'` /
  `hideWhenReferenceClipped` were removed) **in one day, with two breaking changes and no entry for
  either.** Both are exactly what a changelog exists for. **And TT-15 says the default path is
  broken**, so the honest entry cannot be written until that is fixed — do not paper over it.
- **`vue-write-behind` reads 1.0.0** for a package that is days old, has no playground tab, and has
  never been independently audited. 1.0.0 asserts a stability nobody has established. Recommend
  0.1.0 until AUDIT-1 and WBC-3 land; confirm with the owner.
- **`v-dropzone` is still 0.1.0** despite 309 tests and a breaking default change (`clickToPick`).

## What to do

1. **Backfill a changelog for every package**, Keep a Changelog format, honest about what is not
   verified. For the three live packages, reconstruct from git history and npm version metadata —
   `npm view <pkg> time` gives publish dates; do not invent entries you cannot source, mark gaps as
   unreconstructed.
2. **Record today's changes properly.** `v-teleport-to` 2.0.0 and 3.0.0 both need entries naming the
   breaking changes and the migration. `v-copy` 1.1.0 exists — check it covers `dedupe` accurately.
3. **Reconcile versions with reality.** Every version number should mean something a reader can
   check.
4. **Third view on the site.** Renders `CHANGELOG.md` the same way the docs view renders `README.md`
   — rendered from the file, never authored twice. A package with no changelog must render an
   explicit "no changelog yet", not an empty tab.
5. **A missing changelog becomes a publish blocker** in `tickets/_STANDARDS.md`, alongside the
   playground tab and docs view.

## Acceptance

- Every package has a `CHANGELOG.md`; every entry is sourceable from git or npm metadata.
- No entry claims behaviour that AUDIT-1 or TT-15 has not confirmed. **A changelog that documents a
  broken default is worse than a missing one** — it is the documentation lying with a version
  number attached.
- The Pages site shows three views per project, and the changelog view degrades honestly.
- `_STANDARDS.md` updated.

**Ordering, per the owner: DOCS-1 (docs views) → DOCS-2 (changelogs) → the Pages deploy.**
And AUDIT-1 gates the content of every entry.
