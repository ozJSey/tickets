# PG-21 — `cdp.send` has no timeout, so a page reload mid-`evaluate` hangs the run forever

Found by the WBC-3 agent, 2026-09-13, after it bit repeatedly across the session.

`playground/scripts/lib/cdp.mjs` sends a CDP command and awaits the reply with **no timeout**. If the
page reloads while an `evaluate` is in flight — which Vite does on a full-reload whenever an aliased
sibling source changes — the reply never arrives and the whole run blocks indefinitely.

**This is almost certainly behind several incidents this session**: an agent reporting `pnpm
interactions` "sat on check #1 for 20 minutes", the repeated advice that a run "looked contaminated",
and at least two stalled-agent terminations. A hang is also the worst possible failure mode, because
it looks like slow work rather than a fault.

## Fix

- A per-command timeout on `cdp.send`, rejecting with the method name and the elapsed time.
- Detect the reload rather than only timing out: `Page.frameNavigated` / `Runtime.executionContextsCleared`
  arriving while a command is outstanding should fail that command immediately and legibly.
- The WBC-3 agent added a reload detector to one check as a local workaround — generalise it into
  the client so every check inherits it.

## Related harness fragility, same file family

- `scripts/interactions.mjs` contained `` `./demos/*/manifest.ts` `` **inside a block comment**, and
  the `*/` closed the comment — the file would not parse at all. Fixed by the same agent.
- `smoke.mjs` uses `--virtual-time-budget=9000`, too low under load, and reports **0 cards with no
  error** when it runs out (PG-18). It also defaults to port 5199, which collides.

Three separate ways for this harness to fail silently or hang have now been found in one session.
Worth one pass over `playground/scripts/` asking only: *how does each failure mode announce itself?*
