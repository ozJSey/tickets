# WBV-1 — vue-write-behind 0.2.0: four shipped falsehoods, no code defect

**Found by the 2026-09-17 line-by-line audit** — the package was queued to publish as 0.2.0 and was
the only one in that queue with no audit. Its verdict: *"The code is sound… It is not safe to
publish as-is, for three shipped-falsehood reasons rather than any code defect."* The planner
confirmed each of the three directly. **None of this requires touching `src/`.**

## A. `peerDependencies: vue ^3.0.0` — provably wrong

`src/lifecycle.ts:8` imports `getCurrentScope` and `onScopeDispose`; both landed in **Vue 3.2.0**.
On 3.1.5 the built tarball throws at import (`Named export 'getCurrentScope' not found`), and the
esm-bundler entry a Vite/webpack user resolves has no such export either — a build-time failure.

Two places assert the opposite and must be corrected with it:
- `src/lifecycle.ts:27` — a comment stating as fact that *"`getCurrentScope()` exists in 3.0"*. A
  copy-paste consumer reads this file; the comment is part of the product.
- `vitest.workspace.ts:8-10` — claims the 3.3/3.5 matrix exists *"to prove the composable stays
  inside the supported peer range (`vue: ^3.0.0`)"*. It cannot: its lowest version is 3.3.13, more
  than a minor above the first version that has the API. **A gate written so it cannot fail.**
  Either run the real floor in the matrix or delete the claim.

Establish the floor by reading every Vue import, not by assuming 3.2.0.

## B. `npm run check:browser` is dead

`package.json:40` chains `npm run link:core`, which commit `24795b2` deleted along with
`scripts/link-core.mjs`. Confirmed: `npm error Missing script: "link:core"`, and `scripts/` now
holds only `browser-check.mjs` and `lib/`. The underlying script is fine — run directly it passes
**26/26 in headless Chrome against the built dist**. `README.md:338` documents `link:core` too, and
`ARCHITECTURE.md:118-124` has a section on it.

Fix the wrapper and the docs. ARCHITECTURE calls this gate irreplaceable, so leaving it
unrunnable is the worst of both worlds.

## C. The shipped CHANGELOG tells consumers their install will not resolve

`files: ["dist", "CHANGELOG.md"]`, so this ships. Lines 115-117 say:

> **`npm install` needs `@ozjsey/write-behind` to be on the registry.** Until it is, `npm run
> link:core` symlinks the sibling checkout and rebuilds it; the unit suites resolve the engine from
> its source through a vitest alias and need nothing.

All three clauses are now false: the engine **is** live (`npm view @ozjsey/write-behind` → 0.1.0),
`link:core` no longer exists, and the vitest alias was removed in `9ac2ff6`. Written at `5fae16c`,
outlived by the two commits that invalidated it.

**Note for the sweep:** `scripts/changelog-audit.mjs` did NOT catch this — it checks release-state
claims about a package's *own* version, not factual claims about dependencies. Do not extend the
gate speculatively; just record the blind spot in the ticket completion note so DOC-1's owner can
decide.

Also fix `CHANGELOG.md:119` — the `[0.2.0]` release-tag link 404s, as does 0.1.1's.

## D. `ARCHITECTURE.md:95-98` states the opposite of what the tests now do

> *"Both projects alias `@ozjsey/write-behind` to the sibling's **source**, never its `dist/`."*

Since `9ac2ff6` neither project aliases anything — they resolve the published tarball from
`node_modules`, which the audit judged a clear improvement (the tests now exercise the exact bytes
a consumer installs). The docs never followed. This is the single most load-bearing sentence for
"what do the 133 tests prove", and it is inverted.

While there: `tsconfig.json:10-12` points `paths` at `../write-behind/writeBehind.ts`, so
`npm run typecheck` reads a working tree outside this repo and **silently falls back** to
node_modules when it is absent — the same command checks two different type sources depending on
the machine, with no signal. Make it deterministic, and make the two configs agree about which
source of truth they use.

## Explicitly NOT in scope

The audit's P2 test-coverage gaps — `autoFlush` has no test that fails when broken (mutation-tested:
0 failures), nor does the replaced-`ref` arm, nor the `getCurrentScope()` guard. Real, but they are
coverage work, not publish blockers, and this ticket is the blockers. Record them in
`PACKAGE-BACKLOG.md` so they are not lost.

## Version

**0.2.0 stays 0.2.0** — it is unpublished, so these corrections fold into the release that has not
shipped yet. No bump (owner rule: the MAJOR never moves, and nothing here earns even a patch on top
of an unreleased version).

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **A real install check on the declared floor** — install the built tarball against the lowest
   version the corrected range allows and import it; then one minor below, and show it failing.
   That failing half is the negative control.
2. `npm run check:browser` runs end to end and reports its real number.
3. `node scripts/changelog-audit.mjs` clean; every corrected claim checked against
   `registry.npmjs.org`, never against a repo file.
4. The 133-test suite stays green; report the real number.
5. Do not run `npm publish`. Do not commit. Leave the work in the tree.
