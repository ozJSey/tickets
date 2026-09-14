# PUB-3 — publish runbook, 2026-09-15

**You publish. No agent runs `npm publish`** (board decision #5, 2026-09-06). This file is the
exact sequence, in order, with the gate that has to pass between step 1 and step 2.

Measured against `registry.npmjs.org` on 2026-09-15, not against this repo's tables — `TASKS.md`
and `CLAUDE.md` are both stale on at least five packages. `npm whoami` → `ozjsey`.

## Only four packages are behind. Everything else is at parity.

| # | Package | npm today | publishing | why |
|---|---|---|---|---|
| 1 | `@ozjsey/write-behind` | **never published** | 0.1.0 | new engine package |
| 2 | `@ozjsey/vue-write-behind` | 0.1.1 | 0.2.0 | now a ~40-line adapter over the engine |
| 3 | `@ozjsey/v-scroll-into-view` | 1.3.0 | 1.3.1 | SIV-6 — RTL clamp sign belongs to the container |
| 4 | `@ozjsey/v-teleport-to` | 1.1.0 | 1.1.1 | TT-22 — composable measured the pre-render DOM |

Steps 3 and 4 are independent of each other and of 1–2. Steps 1 and 2 are **strictly ordered**.

## The sequence

```bash
# 1 — the engine FIRST. vue-write-behind cannot resolve until this exists.
cd write-behind && npm test && npm run build && npm publish

# --- GATE. Do not skip. ---------------------------------------------------
# vue-write-behind's 133 tests just passed against a SYMLINK to the local
# engine (node_modules/@ozjsey/write-behind -> ../../write-behind), so they
# have never once exercised the tarball npm will actually serve. Prove the
# published artifact resolves before publishing the thing that depends on it:
cd /tmp && npm init -y >/dev/null && npm i @ozjsey/write-behind
node -e "import('@ozjsey/write-behind').then(m=>console.log(Object.keys(m)))"
node -e "console.log(Object.keys(require('@ozjsey/write-behind')))"   # CJS too
# --------------------------------------------------------------------------

# 2 — then the Vue adapter.
cd vue-write-behind && npm test && npm run build && npm publish

# 3 and 4 — independent, either order.
cd v-scroll-into-view && npm test && npm run build && npm publish
cd v-teleport-to      && npm test && npm run build && npm publish
```

`prepublishOnly` is declared on all four, so the test+build above is belt-and-braces — but run it
anyway: `dist/` is gitignored in every package, which means the artifact is only ever as fresh as
the last local build, and stale `dist/` has bitten this repo three times.

## Verified clean before writing this

- **3,315 tests green** across all 12 packages; 12/12 builds; 12/12 `npm pack --dry-run`.
- **No tarball carries a test or spec file.**
- All 12 declare `publishConfig.access: public` — a scoped publish fails without it.
- All 12 are MIT.
- **No package declares a `preinstall`**, so DG-2's supply-chain break (a mandatory
  `dependency-grouper generate` running on the consumer's machine) is not present in anything here.
- `CHANGELOG.md` now ships with all 12 — three of them silently omitted it from `files`, including
  `v-teleport-to`, which is in this batch. Fixed in `8197b41`.

## Known and NOT fixed — your call, does not block publishing

**8 of 12 `repository` URLs 404** (META-1). npm renders `repository` / `homepage` / `bugs` on the
package page, so two of the four releases above — `v-scroll-into-view` and `write-behind` — ship a
dead link today.

| resolves | 404 |
|---|---|
| `big_decimal_string`, `dependency-grouper`, `vue-fit-children`, `vue-write-behind` | `vue-copy`, `vue-dropzone`, `vue-keyboard-navigation`, `vue-observe`, `vue-scroll-into-view`, `vue-select-text`, `vue-teleport-to`, `write-behind` |

`os_ideas` 404s to everyone but you — it is private, so it cannot be the target. Either create the
seven repos, or repoint the dead ones at something public. Ten minutes either way, and it is the
first thing a stranger clicks.
