# PG-20 — `pnpm geometry` only tests one half of the failure space

Found by the `v-fit-children` audit, 2026-09-13. **This is why F1 passes 9/9.**

`geometry.mjs` measures children spilling **past** the host's content edge. That is one of the two
ways this library can fail.

- **Blind to over-hiding.** A rig with **0 of 9 children visible** in a 400px frame reports
  `overflowPx 0` and **PASSES**. An empty row is the symmetric visible failure and the harness
  cannot see it.
- **Never reads the event.** `fit-children-updated` carries `hiddenChildrenCount`, `hiddenData` and
  `isOverflowing`. FIT-1's F1 and F2 are both event defects and are invisible to `geometry` by
  construction.
- **Only drives `input[type=range]`.** Card 03's `keepVisibleEl` checkbox, card 09's `v-show`
  checkbox, card 06's three buttons and card 02's trigger menu are never touched.
- One host per card (`querySelector`), dedupes by title, no console-error capture, fixed 1400×1000
  viewport — so nothing responsive is measured.

**It is not vacuous.** The auditor tried to prove it was and its own negative control refuted it: a
hand-hidden row with one chip too many is caught (`overflowPx 44`, FAIL). The problem is
one-sidedness, not uselessness.

## Fix

- Assert the **lower** bound too: given a host wide enough for N children, fewer than N visible is a
  failure. That single check would have caught F1.
- Read the event alongside the geometry and cross-check it against the measured reality — the
  attribute and the event disagreeing is exactly FIT-1's F2.
- Drive checkboxes and buttons, not only sliders.
- Capture console errors.
- Sweep at least one narrow viewport; six of nine cards clip at 375px (that one is the card stage,
  not the library — see FIT-1 F8 — but the harness should still see it).

**And say what it covers.** "9/9 demos laid out correctly" reads as a much broader claim than
"no child spilled past its host at these slider positions on a 1400px desktop".
