# OBS-2 — `viewed` (IAB dwell) and `stuck` (sticky pinned-state)

**Owner, 2026-09-16 (audit walkthrough, stop 10):** build both. This ticket takes the
**intersection path** only, so it runs disjoint from OBS-1 (mutation path). One agent, two features
that share the intersect option surface and the `data-observe-state` hook.

## A. `viewed` — the viewable-impression state machine

Every analytics or ads integration needs *"fire when 50% visible for 1 continuous second, on a
focused tab"* — the IAB viewable-impression standard — and every team hand-rolls the timer, forgets
tab-visibility pausing, and forgets to reset the clock when the ratio dips.

**The package's own README recipe currently gets this wrong**: it fires on the first 50% tick with
zero dwell. Fixing that recipe is part of this ticket.

```html
<Ad v-observe="{ intersect: { viewed: { on: trackImpression } } }" />
<!-- defaults: for: 1000, ratio: 0.5, continuous clock, paused while the tab is hidden, fires once -->
<Card v-observe="{ intersect: { viewed: { for: 3000, cumulative: true, on: markRead } } }" />
```

- Continuous by default; `cumulative: true` accumulates instead of resetting.
- **Paused while `document.visibilityState` is hidden** — a backgrounded tab must not accrue dwell.
  This is the part hand-rolled versions miss, so it gets its own test.
- Clock resets when the ratio drops below `ratio` (continuous mode).
- Fires once, then stops observing.
- Writes `intersect:viewed` into `data-observe-state` for CSS "already seen" styling.
- Absorbs the brief's unshipped `stableFor` item — note that explicitly in the completion report so
  the backlog entry is closed rather than orphaned.

## B. `stuck` — native `position: sticky` pinned-state detection

No native event exists, and the IntersectionObserver workaround (threshold `[1]` plus an
inset-compensated `rootMargin`) is fiddly enough that people fall back to scroll listeners.

```html
<header style="position: sticky; top: 0"
        v-observe="{ intersect: { stuck: (e) => pinned = e.stuck } }">…</header>
```

```css
header[data-observe-state*='intersect:stuck'] { box-shadow: 0 2px 8px rgb(0 0 0 / .15); }
```

- Read the element's **computed** `top`/`bottom` inset and build the compensated observer from it;
  do not require the consumer to supply a magic offset.
- Emit `{ stuck, edge }` and write `intersect:stuck` into the state attribute — the CSS-hook half is
  what no incumbent offers (the vanilla technique itself is public, so the hook is the wedge).
- Zero scroll listeners. If the element is not `position: sticky`, warn — that is a consumer bug.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **Real-browser checks for both** — jsdom implements neither layout nor sticky, so unit tests
   here prove plumbing only and must be reported as such.
2. `viewed` dwell check driven by real scrolling: below-threshold → no fire; held above for `for`ms
   → fires exactly once; dipped mid-clock → no fire (continuous), fires (cumulative).
3. **Tab-hidden pause check**: background the tab mid-dwell via CDP, assert the clock did not accrue.
   **Negative control:** remove the pause, watch it fire — this is the failure mode the feature exists for.
4. `stuck` check measures the state attribute flipping at the real pin point, both edges (`top` and
   `bottom` sticky). Negative control: non-sticky element warns and never reports stuck.
5. README recipe 3 corrected in the same pass, with its playground card link (DOCS-4 convention).
6. Release: **minor** (two real capabilities); the major position does not move.
