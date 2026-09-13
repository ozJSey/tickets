# TT-17 — **P0.** The fit test never runs when `v-show` is written after the directive

Found by the **blind re-audit**, 2026-09-07 — in the fix that closed TT-15 hours earlier.
**Reproduces against `dist`.**

## Measured

Two hosts, identical binding `{ to: trigger }`, differing only in source order. 1280x482, trigger
at `top: 200px` (200px above, 250px below), menu = 12 rows x 28px.

```
<div v-teleport-to="{to:trig}" v-show="isOpen">    ← THE README'S USAGE EXAMPLE
   opens 1,2,3:  placement=top  fit=unmeasured  max-height=200px
                 rendered 200, scrollHeight 336  ->  136px of the menu cut off

<div v-show="isOpen" v-teleport-to="{to:trig}">
   opens 1,2,3:  placement=bottom  fit=flipped  max-height=240px  (whole menu)
```

## Cause — Vue-level, not a playground artifact

`compileTemplate` emits the directive array in **source order**:
`[[_directive_teleport_to, opts], [_vShow, isOpen]]` versus the reverse. When the directive runs
first, `v-show` has not yet cleared `display: none`, so `measureHostExtent` returns `null`,
`resolvePlacement` reports `fit: 'unmeasured'` and keeps the comparative side, and `max-height` is
clamped to the cramped side. **It stays wrong on every subsequent open** until an unrelated scroll,
resize or parent re-render.

## Why nothing caught it

**All three `v-show` cards (01, 02, 09) write `v-show` first. The README writes it second.** The
TT-15 fix was verified in a shape that differs from the documented shape.

## TT-2 — the composable recipe fails the same way, and its documented remedy does not work

README "Composable: useTeleportTo" recipe pasted verbatim, same geometry:
```
host passed (the recipe)      -> fit=unmeasured  placement=top     max-height=200px  (336 content)
host omitted                  -> fit=unmeasured  placement=top     max-height=200px
host passed, always rendered  -> fit=flipped     placement=bottom  max-height=240px
after a window resize         -> fit=flipped     placement=bottom  max-height=240px
```
Passing the host — which the README presents as the fix for degradation — buys **nothing** on the
documented shape, because `watchEffect(..., { flush: 'sync' })` runs before the DOM shows the host.

## Fix — the measurement must not depend on directive ordering

The host being `display: none` at the moment the directive runs is not an error state; it is the
normal condition of a dropdown that has never been opened. Options to weigh:

- Re-measure on the next frame when the first measurement fails, rather than committing
  `'unmeasured'` permanently.
- Detect the host becoming visible (its own `IntersectionObserver`, or a `MutationObserver` on
  `style`) and recompute once.
- Measure off-screen before `v-show` paints — but note the host may legitimately have no box.

**Whatever is chosen must hold for both orderings and for the composable's `flush: 'sync'` timing.**

## Acceptance

- **A regression that fails in the README's order before the fix.** Both orderings must produce
  identical placement, `fit` and `max-height`.
- The composable recipe pasted verbatim produces `fit: 'flipped'` on the first open.
- Add a card that writes `v-show` **after** the directive — the ordering the README teaches and no
  card exercises.
- Verify against `dist`; the re-audit reproduced it there.
- 806 tests stay green.
