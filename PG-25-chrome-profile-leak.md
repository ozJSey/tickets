# PG-25 — every browser gate leaks a Chrome profile directory

**Owner, 2026-09-16:** *"Obvious one yes? We need to fix that leak."*

## Measured

19 orphaned `dz-cdp-*` directories under `$TMPDIR`, **567 MB**, oldest 2026-09-15 01:08 — one per
run of `interactions` / `smoke` / `geometry` / `standalone`, left on **every** exit path including
clean ones. Source: `playground/scripts/lib/cdp.mjs:198` creates the profile with
`mkdtempSync(join(tmpdir(), 'dz-cdp-'))`; `grep -n 'rmSync\|rm -rf'` over `cdp.mjs` + `watchdog.mjs`
returns nothing. The PG-21 watchdog reaps **processes**, not **directories**.

## Fix — in the teardown the watchdog already owns

1. `launchChrome` registers `userDataDir` alongside the process it registers today.
2. `watchdog.mjs`'s exit cleanup (`reapOwnChildren` / `installExitCleanup` paths) removes every
   registered dir with `rmSync(dir, { recursive: true, force: true })` — **after** the process
   tree is killed, never before (Chrome still writing → races). Synchronous, exit-handler-safe.
3. **Stale sweep for the unfixable path:** a SIGKILL of node itself runs no exit handler, so
   `launchChrome` also sweeps `dz-cdp-*` dirs older than 24h at startup. Self-healing, bounded.
4. One-time: remove the 19 existing orphans (the sweep in 3 does this on first run).

## Acceptance — standing criteria (BOARD.md), all agents at xhigh effort

- Existing negative controls still pass: `node scripts/lib/watchdog.mjs` and
  `node scripts/lib/cdp.mjs` (all modes) — extend the watchdog control to also assert
  **the profile dir is gone** after the wedge fires (it already asserts the child is reaped).
- Stale-sweep control: plant a fake `dz-cdp-` dir with an old mtime, launch, assert swept;
  plant a fresh one, assert untouched.
- End-to-end: count `dz-cdp-*` before and after one `pnpm smoke` run — delta must be 0.
