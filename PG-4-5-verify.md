# PG-4/PG-5 — verify the harness fixes (code-complete, never run)

**State:** an agent wrote this code then died to a session limit before running anything.
Do not assume it works. **First job at reset.**

**What is already in `playground/scripts/interactions.mjs`:**
- `stage(file)` helper returning `.demo__stage`; `button()` and `label()` route through it.
  (Fixes the bug where `__pg.button(file, 'Copy')` matched `DemoCard.vue`'s own
  copy-the-source button in `.demo__head`, which precedes the stage in document order.)
- `readManifests()` reading `playground/src/demos/*/manifest.ts` for the true denominator.
- UNCOVERED reporting + a coverage summary.
- `playground/scripts/interactions-coverage.json` — ratchet at `librariesWithSpec: 2`,
  `demoCardsWithChecks: 24`.

**Verify:**
```
cd playground
pnpm interactions              # both existing specs still green (74 checks)
PLAYGROUND_TARGET=dist pnpm interactions:dist
pnpm smoke && pnpm typecheck
```
**Acceptance:** 74/74 still pass; the run no longer ends on a bare `74/74 passed` — it must
name that 5 of 7 libraries are uncovered and 59 of 83 cards have no check; the ratchet fails
the run when lowered artificially; no existing check was relying on the old buggy scoping
(if one was, report it, do not preserve the behaviour).

**Note:** if PG-11 (Playwright migration) is approved, PG-4 is superseded by strict locators
and PG-5 by the ledger. Verifying now is still correct — it is the current gate.
