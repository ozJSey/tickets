# TT-23 — `topLayer: true`: native top-layer escape, superseding the v2.0 reparenting plan

**Owner decision, 2026-09-16 (audit walkthrough, stop 5): topLayer REPLACES the v2.0 rewrite.**
The planned breaking DOM-reparenting rewrite in `instructions/v-teleport-to.md` existed for one
problem — a `transform`ed ancestor hijacks the containing block, so "escapes all clip containers"
fails exactly there. The platform now solves it natively; a rewrite that breaks consumers to solve
what one opt-in key solves fails the portfolio's own uniqueness bar against the platform itself.

## Design

- New option `topLayer: true` (default **false** — current behaviour is untouched without it).
- While the binding is enabled: `el.popover = 'manual'; el.showPopover()`. While disabled:
  `hidePopover()` (guarded — it throws on an element not in the top layer). Promotion to the top
  layer gives the element the **viewport** as containing block regardless of ancestor
  transform/filter/contain — and it never leaves its DOM position, so scoped styles, inherited
  custom properties, Vue event bubbling, focus order and HMR keep working. Zero reparenting
  bookkeeping.
- The math is untouched: the directive already emits viewport-relative fixed coordinates, which is
  exactly what the top layer resolves against.
- **Feature-detect** `'showPopover' in el`. Absent (older browsers, jsdom): today's fixed-position
  path, byte-identical. Baseline: Chrome 114+, Safari 17+, Firefox 125+ (2024).

## Named risk area — the UA `[popover]` stylesheet

The implementer must measure, not assume, the interactions with:
- UA resets on `[popover]`: `margin`, `inset`, `border`, `background`, and
  `display: none` via `:not(:popover-open)` — this collides head-on with the
  `data-teleport-state` CSS animation recipe and the `enabled` + `v-if` pairing card 11 pins.
  The per-tick style writes must fold in whatever the UA sheet would otherwise override.
- `::backdrop` exists once promoted — leave it untouched, document that it is the consumer's.
- Dormant/teardown paths: `clearPositionAttributes` learned this lesson once (TT-22 §6) — the
  popover attribute and top-layer state must be cleared by the same shared helper discipline.

## Supersession task

Mark the v2.0 reparenting P0 brief in `instructions/v-teleport-to.md` **superseded by TT-23**,
pointing at this ticket and the decision — do not delete the analysis, annotate it. The README's
transformed-ancestor limitation section becomes the `topLayer` documentation.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. Real-browser card: the documented failing case verbatim — reference inside a
   `transform: translateZ(0)` ancestor — measured clipped without `topLayer`, escaped with it.
2. **Negative control:** feature-detect forced off on the same card reproduces today's clipping
   byte-identically (same measured rects as the shipped 1.1.1).
3. jsdom units cover option plumbing and the guarded hide path only; jsdom has no `showPopover`,
   so the fallback branch is the jsdom-natural one — top-layer behaviour is browser-checked only
   and reported that way, never claimed from jsdom.
4. README: support-baseline stated plainly; the animation-recipe section updated for the UA
   `display` semantics; no claim that `topLayer` is needed on markup that already works.
5. Version: **patch-level** — owner rule 2026-09-16: no generous minors; opt-in additive key on a
   package released this week.
