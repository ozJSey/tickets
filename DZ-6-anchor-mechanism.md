# DZ-6 — replace the inline `position: relative` write with a zero-specificity rule

Deferred with reason by the DZ-5 fix, which documented the current behaviour precisely instead of
swapping the mechanism under a P0 fix. **Two defects, not one:**

1. A conditional `position` (e.g. `sticky` under a media query) is **permanently killed** — inline
   beats any stylesheet and nothing removes it.
2. A resize with **no re-render** leaves the zone *unanchored*, so **DZ-3's Tab-scroll jump is live
   in that window.**

The README now carries the exact four-row table and the one-line consumer fix (declare `position`
unconditionally). That is honest, not a solution.

## The proposed mechanism

A zero-specificity rule — `:where([data-dropzone-anchor]) { position: relative }` — fixes all four
rows with no listener, no re-render dependency, and **retires the string-`:style` hazard too** (Vue
patches string styles with `el.style.cssText = next`, which wiped the inline write and put DZ-3
straight back; that is currently handled by re-asserting on every update).

## Why it needs its own ticket rather than riding along

This is the module that closed a **P0** two runs ago. It carries ~15 unit assertions on the inline
write and a measured six-host-shape table. Swapping the mechanism means re-verifying all of it.

**Required controls:** shadow DOM (does the rule reach a host inside one?); a consumer stylesheet
that also matches the host; `position` set inline by the consumer; hosts already `absolute`,
`sticky`, `fixed`, or inside a transformed ancestor; and the string-`:style` case that broke it before.

## Acceptance

- All four rows of the DZ-5 table behave correctly, including the unanchored-after-resize window.
- The re-assert-on-update workaround for string `:style` can be **deleted**, not merely kept.
- DZ-3's negative control still fails without the fix: strip the mechanism and `scrollY` must jump
  (measured previously at 4,754px and 8,423px on two different pages).
- 330 tests stay green; the six-host-shape measurements are re-run, not assumed.

---

## PARTIALLY ADDRESSED — 0.1.1, 2026-09-13. **The mechanism swap is still open.**

0.1.1 fixed a *third* defect in this module that the quality audit found, and it is worth
separating from the two above so this ticket is not read as closed:

**`releasePickerHost` removed a `position` the consumer wrote.** `pickerHostPositioned` recorded
that the directive had written a position at *some point*, not that the write was still standing.
`anchorPickerHost` returns early on an already-positioned host and left the flag set, so once a
consumer positioned the host themselves the flag still said "mine" — and the next
`teardownClickToPick` (reactive `clickToPick: false`, or `enabled: false`) deleted their value,
with Vue declining to put it back. Teardown now also checks that the current inline value is still
the exact string this module writes. `ARCHITECTURE.md:56` no longer claims the flag alone does
this.

**Still open, unchanged:** both defects this ticket was filed for.

1. A conditional `position` (e.g. `sticky` under a media query) is still permanently killed by the
   inline write — inline beats any stylesheet.
2. A resize with no re-render still leaves the zone unanchored, so DZ-3's Tab-scroll jump is live
   in that window.

The `:where([data-dropzone-anchor]) { position: relative }` proposal, the six-host-shape re-measure
and the deletion of the re-assert-on-update workaround all remain as written above. **Deferred out
of 0.1.1 deliberately**: swapping the mechanism means re-verifying ~15 unit assertions and a
measured six-host table, in the module that closed a P0 two runs ago, inside a patch whose subject
was the upload state machine. That is two unrelated risks in one release.
