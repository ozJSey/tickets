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
