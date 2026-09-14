# CI-1 — daily CI on every repo, and a playground that tests the *published* packages

Owner, 2026-09-15: *"playground must have a github.ci to fetch these daily, each repo must have
their ci test runs daily, all should be using npm endpoints directly."* Then, on cadence:
*"we need to fetch like 2-3 times a day to make sure all dependencies are up to date. Otherwise we
cannot maintain all 11 like that."*

**Cadence: every 8 hours** (`cron: '0 */8 * * *'` — 3 runs/day), not daily. The reason is
maintenance load, not paranoia: 12 packages with independent lockfiles cannot be watched by hand,
so CI has to be the thing that notices a transitive dependency moved. Three runs a day means a
break is at most ~8h old when it is found, and the failing run names the package.

This is the first CI in the portfolio (board decision #10 said "after publishing" — publishing is
now happening, so it is due).

## Two different jobs, and the second is the valuable one

**A — per-package CI, 3x daily.** Each of the 12 package repos gets `.github/workflows/ci.yml`:
`npm ci` → `npm test` → `npm run build` → `npm pack --dry-run`, on push, on PR, and on a daily
cron. Catches bitrot in transitive deps and in the toolchain, which is the only thing that can
break a package nobody is editing.

**B — playground CI, 3x daily, against the registry.** `npm-portfolio-playground` installs the
**published** `@ozjsey/*` packages from npm — no source alias, no `file:` link, no workspace — and
runs the browser suites against them. This is the only gate in the portfolio that sees what a
consumer sees.

Why B matters more than A: every existing gate tests the source tree. The playground's default
target aliases each specifier to the sibling's `.ts`; `PLAYGROUND_TARGET=dist` tests the local
build. **Neither has ever tested a tarball that came off the registry.** Two defects this month
were invisible to the source tree and would have been caught here:

- `bigdecimal-string` 1.1.0 shipped with no `import` condition, so ESM consumers could not load it
  at all. Every local suite was green.
- `smoke:dist` silently falls back to **source** for any scoped package (PG-15), so the dist gate
  has been a no-op for the packages that are actually published.

## Requirements

1. `npm ci` from the registry — pinned by a committed lockfile, no aliasing.
2. `schedule:` cron every 8 hours on both A and B, plus `workflow_dispatch` to run by hand.
3. B must fail loudly when a published package is broken, and must name **which** package — PG-22's
   lesson: one bad package used to blank every tab and the runner said only "no tabs rendered".
4. B runs the real browser suites (`smoke`, `interactions`, `geometry`), not just a build.
5. **The load gate (PG-24) must not fire spuriously on a CI runner.** A 2-core GitHub runner under
   its own test load can sit above the 50% threshold legitimately. Decide deliberately: either set
   `BUSY_MAX_CPU` for CI, or gate on `process.env.CI`. Do not discover this as a red build.
6. No npm token in A or B — both are read-only against the public registry. Publishing stays
   manual and stays the owner's.

## Open question for the owner

The playground is deployed to GitHub Pages from `npm-portfolio-playground`. If B installs from the
registry, **the deployed site starts showing the published packages rather than the working tree.**
That is almost certainly what you want for a docs site — it stops the docs promising things that
are not released — but it is a behaviour change and is worth saying out loud before it ships.
