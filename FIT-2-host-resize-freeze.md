# FIT-2 — P0: a host-only resize round-trip freezes the row forever, while the state attribute says "fits"

**Live on npm as `@ozjsey/v-fit-children` 2.3.0 (`latest`).** The published `dist` is
byte-identical to a fresh local build, so every consumer has this today. Found by the 2026-09-16
audit fleet; **structurally confirmed by the planner** before dispatch.

## The defect, confirmed in the source

`schedule.ts:68` records a run as "proven too big" whenever a fresh fit is smaller than the applied
run:

```ts
if (state.visible.size > fit.visible.size && state.visible.size > 0) { … record … }
```

That condition is also satisfied by a **genuine external shrink of the host** — and the host is the
one path that never clears the record. In `observers.ts` the container branch clears
`oversizedRuns` (line 41), the parent branch clears it (53), the sibling branch clears it (69) —
the **host branch (35–36) sets `relevant = true` and clears nothing.**

Sequence: mount at 300px with 3 chips fitting → the host alone narrows to 150px (sidebar, splitter
drag, class toggle; parent and siblings unmoved) → the 3-chip run is recorded against
`appliedAvailable = 300` → the host returns to exactly 300px → `computeFit` says all 3 fit, but
`provenTooBig` matches (`300 <= 300 + EPSILON`) and lines 81–88 re-apply the 1-chip run.

**Result:** 1 of 3 children shown forever at a width where all three fit; `data-v-fit-state` reads
`"fits"`; the dispatched event says `isOverflowing: false` with `hiddenChildrenCount: 2`. That is
the attribute/event contradiction class 2.3.0 was published to eliminate. Repeated splitter drags
accumulate up to 4 frozen widths. **Nothing recovers a static row** — host entries never clear,
hidden children emit no RO entries, and the frozen pass's own un-hide/re-hide nets to no entry.

Auditor's repro, against source: after hostWidth 300→150→300 with only host RO entries,
`expect(visibleCount(host)).toBe(3)` fails with `expected 1 to be 3` — while the preceding
`expect(host.getAttribute("data-v-fit-state")).toBe("fits")` **passes**.

## Fix direction (implementer decides, with measurement)

The guard must distinguish **"our own output fed back"** from **"the world genuinely changed"** on
the host path. Two candidates named by the audit:
- clear `oversizedRuns` when a host entry's width differs from the last measured `hostWidth` while
  `state.lastApplyMoved` is false; or
- expire records after a pass that changed nothing.

Do not simply delete the guard — read the comment block above line 68 first. It exists because a
badge that widens as a consequence of hiding is a real feedback loop (the documented
"can settle below the theoretical maximum" limitation). **Whichever fix lands must keep that loop
closed**, and the existing loop tests must stay green — this is a narrowing, not a removal.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. The repro above as a unit test, red-before/green-after, asserting **both** the visible count
   and that no state attribute/event pair ever disagrees.
2. **Real-browser check** (`pnpm interactions`): a card whose host alone is resized down and back
   — chips return. jsdom cannot see layout, so the unit test proves plumbing and the browser check
   proves behaviour; report them separately.
3. **Negative control:** revert the fix, watch exactly those two go red.
4. The existing badge-feedback-loop tests must stay green — name them in the completion note.
5. Default configuration covered (standing rule): bare `v-fit-children` with no options is the
   repro's binding.
6. Release: **2.3.1 patch** (owner rule 2026-09-16 — major never moves). Carries FIT-3's
   CHANGELOG correction in the same publish.
