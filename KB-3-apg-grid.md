# KB-3 — APG grid: full 2D keyboard navigation, from structure, never geometry

**Owner, 2026-09-16 (audit walkthrough, stop 9): build it.** This is the deliberate reversal of the
earlier deferral, whose reasoning stands and becomes this ticket's constraint: *a half-done grid is
the worst outcome.* **Ship the whole APG grid pattern or ship nothing.**

## Why this package is the right home (measured, from DESIGNS.md → KB-1 and the ideation pass)

Emoji pickers, avatar walls, calendars and transfer tables are the most common composite widgets
with **no Vue directive answer** — today you restructure into Ark's `createGridCollection` or
hand-roll ~200 lines. Reka's `RovingFocusGroup` is `vertical | horizontal` only. tabster's grid
moves to the "visually adjacent" item — **geometry-driven**, which means a resize silently changes
what a key does. The focusgroup explainer excludes grid from V1. VueUse has nothing.

And the package's measured wedge doubles here: a 2D scroll pane needs the
`focus({ preventScroll: true })` → `block: 'nearest'` ordering on **both** axes, which nobody ships.

## Non-negotiable: rows come from structure, never from measurement

- `role="row"` / `role="gridcell"` structure when the markup has it (tables, calendars).
- An explicit `columns: N` for a flat wrap-flow grid (emoji picker).
- **Never `getBoundingClientRect`.** Geometry-derived adjacency means the keyboard behaves
  differently at different window widths — the exact anti-pattern tabster demonstrates. If the
  structure is absent and `columns` is unset, warn and fall back to linear; do not guess.

## The complete pattern (all of it, or the ticket is not done)

- ArrowLeft/Right within the row, ArrowUp/Down across rows, respecting RTL for the horizontal axis.
- Home / End → first / last cell **in the row**; Ctrl+Home / Ctrl+End → first / last cell in the grid.
- PageUp / PageDown → by visible rows (derived from the scroll container's client height and row
  count, not from per-cell geometry).
- Ragged final rows, `colspan`/`rowspan` where `role="row"` structure declares them, and skipped /
  `aria-disabled` cells excluded from traversal — `items.ts` already knows those piles; reuse it.
- Scroll on both axes via the existing preventScroll-then-nearest path.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **Real-browser checks** covering every key in the list above, on two cards: a `role="row"` table
   and a flat `columns: N` wrap grid. Read focus and scroll position back out of the live DOM.
2. A **2D scroll check**: a grid larger than its pane, asserting the 1-item follow on both axes
   (the wedge) rather than the UA's centring lurch.
3. Ragged-row and disabled-cell traversal pinned explicitly — these are where hand-rolled
   implementations break.
4. **Negative control:** with the grid role removed, the same cards behave linearly — proving the
   grid path is what the checks are exercising.
5. RTL horizontal traversal checked, not assumed.
6. Report anything unproven (screen-reader behaviour in particular) as **UNPROVEN**, per the
   standing rule — never counted as passing.
7. Release: **minor** — this is a genuine new capability, and the owner's rule permits minors for
   real features. The major position does not move.
