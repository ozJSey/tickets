# TT-24 — `trigger`: the bare directive becomes a complete popover

**Owner, 2026-09-16 (audit walkthrough, stop 13):** approved alongside TT-23. `debug` parked, not killed.

**Sequencing: runs AFTER TT-23 (topLayer), and after v-teleport-to's own audit lands.** Both
tickets touch the `enabled` / state path, and that audit died on the session limit. Do not dispatch
this concurrently with TT-23.

## The gap this closes

Every consumer hand-rolls the same four things around this directive: an `open` ref, a click
handler, outside-click + Escape dismiss, and `aria-expanded`. The README's **biggest documented
footgun** — forgetting `enabled` beside `v-show` — exists *only* because the consumer owns open
state. Move the state machine in and the footgun disappears with it.

This is the portfolio's "bare directive covers 95%" rule applied to the half of the problem the
package currently leaves on the consumer's desk.

## Design

```html
<button ref="trigger">Menu</button>

<!-- no open ref, no @click, no onClickOutside, no aria wiring -->
<div v-teleport-to="{ to: trigger, trigger: 'click' }" class="menu">…</div>

<!-- hover with safe-polygon intent and a close delay -->
<div v-teleport-to="{ to: card, trigger: 'hover', closeDelay: 150 }" class="preview">…</div>
```

- `trigger: 'click' | 'hover' | 'focus'` toggles `enabled` **internally**. An explicitly-passed
  `enabled` still wins — a consumer who wants control keeps it, and the two must not fight;
  document the precedence.
- Drives the **existing** `data-teleport-state` contract, so the README's CSS animation recipe
  finally runs without the consumer wiring anything.
- Dismiss on outside `pointerdown` and on Escape; **return focus to the reference on close**.
- Stamps `aria-expanded`, `aria-controls`, `aria-haspopup` on the resolved reference — and clears
  them on teardown. The reference is not the directive's element, so cleanup must reach outside the
  host: same class of bug TT-22 §6 found (a dormant branch hand-listing attributes). Use one shared
  clear helper.
- **`hover` ships with hover-intent via a safe polygon** so the cursor can travel diagonally from
  the reference to the host without the popover closing underneath it. Without this, `hover` is
  worse than not shipping it — it is the whole reason Floating UI users reach for React's
  `useHover`. Treat a naive mouseleave implementation as a failed ticket.
- Interaction with TT-23: when `topLayer` is on, open/close must also drive
  `showPopover()`/`hidePopover()` in the right order relative to the state attribute writes.

## Honest positioning for the README

V-Float (a third-party Vue port of Floating UI) ships `useHover` with `safePolygon`, `useClick` and
`useDismiss` — as **composables**, built on Floating UI packages. The beat is the entry point (one
key on markup you already wrote), zero dependencies, and feeding the existing state/ARIA contract.
Say that; do not imply nobody has these behaviours.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **Real-browser checks** per trigger mode: click opens/closes; outside pointerdown dismisses;
   Escape dismisses **and focus returns to the reference** (assert `document.activeElement`).
2. **Safe-polygon check:** drive a diagonal cursor path from reference to host through the gap via
   CDP and assert the popover stays open. **Negative control:** disable the polygon, same path,
   watch it close — this is the feature's whole value and needs a check that can catch its absence.
3. ARIA attributes asserted on the **reference** while open, and gone after teardown.
4. Precedence check: explicit `enabled` beside `trigger` behaves as documented.
5. Combined with `topLayer: true`, all of the above still hold (one card exercising both).
6. Default configuration untouched (standing rule): bare bindings with no `trigger` behave exactly
   as today, proven by the existing cards staying green.
7. Release: **minor**; the major position does not move.
