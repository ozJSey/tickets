# OBS-1 — four `v-observe` demo cards have no browser check

Opened 2026-09-13, at the end of the `v-observe` audit close-out (0.2.0). The package's first
browser spec landed in the same run — `playground/scripts/interactions/v-observe.mjs`, 19 checks
over 13 of the 17 cards. These four are what it does not reach.

Nothing here is a known defect. It is uncovered surface, on a package whose entire audit history
says jsdom cannot see its bugs: every one of the four features below is verified today only by unit
tests driving mocks the suite itself constructs, which is exactly the arrangement that let `root`,
`box` and the `attr:*` freeze ship.

## The four

**`03-direction.vue` — scroll-direction inference.** `enter-from-above` / `enter-from-below` /
`leave-to-above` / `leave-to-below` are inferred by comparing `boundingClientRect.top` between two
callbacks, with a first-tick fallback that compares the element's top against the root's centre
(`src/intersect.ts`, `inferDirection`). Both halves depend on the sign of a real layout delta and on
`rootBounds` being the pane rather than the viewport, and neither exists in jsdom. Drive it by
scrolling the demo's pane down past the probe and back up, and assert all four directions appear in
the right order. The interesting case is the **first** tick — the element that is already partly in
view when the observer attaches — because that is the branch with the invented `rootCenter`.

**`08-resize-crossed.vue` — `on: 'crossed'` with `axis: 'both'`.** The unit suite covers the
crossing maths thoroughly, including the per-crossing bracket labels fixed in 0.2.0. What it cannot
cover is that a single real resize which crosses thresholds on *both* axes at once produces one
event per axis per threshold, in a sane order, off one `ResizeObserver` callback. Set the demo's box
from 200×200 to 800×800 in one assignment and read the log.

**`14-mutate-removed.vue` — self-removal.** `on: 'removed'` attaches a second `MutationObserver` to
`el.parentNode` recorded at mount, and auto-disconnects after firing once. The unit suite fires that
observer by hand. jsdom's real `MutationObserver` now covers the basic case
(`vObserve.test.ts`, "a real self-removal fires `removed` exactly once"), but the demo's scenario —
a third party yanking the host while Vue still thinks it is mounted, then Vue unmounting later — has
never run in a browser. Check that `removed` fires exactly once and that unmounting afterwards does
not throw.

**`15-mutate-multi.vue` — array `on` + debounce + the `mutate:active` hook.** The merge rules
(first `from`, latest `to`, concatenated `added`/`removed`) and the 150 ms `active` → `idle` cooldown
are timing behaviour over a real observer's batching, which is precisely where the mock diverges: a
real `MutationObserver` decides its own batch boundaries. Fire a burst of mixed mutations inside one
debounce window and assert one event per type, with the merged payloads, and that
`data-observe-state` shows `mutate:active` during the window and `mutate:idle` after it.

## Acceptance

- Four checks added to `playground/scripts/interactions/v-observe.mjs`, one per card.
- Each one **negative-controlled**: break the behaviour in `v-observe/src/`, confirm the check goes
  red, restore. Say in the commit which line you broke for each. A check that stays green with the
  feature removed is not a check.
- `scripts/interactions-coverage.json` raised to match (v-observe reaches 17/17).
- Run it as `PORT=<your own> CDP_PORT=<your own> ONLY='(03-direction|08-resize-crossed|14-mutate-removed|15-mutate-multi)\.vue' node scripts/interactions.mjs`.
  Do not use port 5199: `cdp.mjs` has no send timeout (PG-21), so a Vite full-reload mid-`evaluate`
  hangs the run forever.
- Read `playground/scripts/interactions/v-observe.mjs`'s header first. Two traps cost an hour each
  in the run that wrote it: `__pg.txt()` collapses newlines, so a `pre.pg-log` read is a single line
  (match with regexes, never `split('\n')`); and check `fn`s are stringified plain functions, so
  regex escapes are written once (`\d`), while the prelude is a template literal and needs them
  doubled (`\\d`).
