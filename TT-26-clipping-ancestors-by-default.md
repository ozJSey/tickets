# TT-26 — detect the clipping/scroll ancestors instead of asking for them

**Owner, 2026-09-20:** *"I see when it's hidden, we still show it, boundary (first scrollable
container) should be the default I feel, it looks awful when I scroll and teleport-to survives
scroll. Perhaps a full refactor is needed there I wanna say our problem is chronic and we couldn't
solve it so far."*

The diagnosis is right, and the "chronic" is right too: this family of bugs has one root.

## The root cause, in two defaults

Both are documented, both are individually defensible, and together they produce exactly what the
owner saw — a popover inside a scrollable panel that neither moves nor hides when the panel
scrolls.

1. **`scrollContainer` defaults to `window`** (`src/scroll-target.ts`). It resolves listeners from
   an EXPLICITLY NAMED container; there is no ancestor walk. An inner pane's scroll therefore
   fires no listener at all, so nothing recomputes.
2. **`boundary` defaults to `'viewport'`** (`src/types.ts:212`), and the reference-hidden predicate
   is `clip = intersect(boundaryRect, viewportRect)` (`src/calculate-position.ts:597-610`). With
   the default boundary, `clip` IS the viewport — so a reference that leaves an inner pane while
   remaining inside the viewport is not "hidden", and `hideWhenReferenceHidden` (on by default,
   `calculate-position.ts:641`) cannot fire.

Verified there is no clipping-ancestor detection anywhere: `offsetParent` appears only as the
coordinate origin for `strategy: 'absolute'`, and no module reads `overflow` off an ancestor.

**So the library asks the consumer to declare what it could detect.** That is the chronic part.
TT-17, TT-18 and TT-19 were all "the measurement was taken against the wrong box"; this is the same
shape one level up — the CLIP is taken against the wrong box, because nobody told it about the box.

## The fix — one detection, two consumers

Walk the reference's ancestors once and collect every element with a scrolling/clipping `overflow`
(`auto`, `scroll`, `hidden`, `clip`), up to the document. Then use that ONE chain for both:

- **scroll listeners** — the popover repositions when any ancestor scrolls, not only `window`
- **the clip** — `clip = intersection(every clipping ancestor) ∩ viewport`, which is what visual
  clipping actually is. The current single-`boundary` intersection is a special case of it.

This is the shape mature popover libraries converge on (Floating UI attaches `autoUpdate` to
overflow ancestors and clips its `hide` middleware against `clippingAncestors`), and it makes the
correct behaviour the DEFAULT rather than a configuration the consumer has to know to reach for —
which is the owner's standing smart-defaults rule.

`boundary` and `scrollContainer` stay as explicit overrides. They stop being the only way to get
correct behaviour and become what they read like: overrides.

## Constraints

- **Patch-level, not a rewrite.** "Full refactor" is the owner's framing of the symptom; the change
  is one new module plus two call sites. Resist the urge to rewrite `calculate-position.ts`.
- **Cost:** the walk is O(depth) and happens on resolve, not per tick. Cache per reference and
  invalidate when `to` changes. Do NOT walk inside the measurement path.
- **`overflow: hidden` counts.** It clips visually even though the user cannot scroll it, and
  `v-scroll-into-view/src/scrollers.ts` already learned this: "Chrome scrolls `overflow: hidden`
  boxes too (they are scrollable programmatically, just not by the user)."
- **A browser check is mandatory**, and jsdom cannot host it: jsdom has no layout, so every rect is
  zero and the wrong clip and the right one agree exactly. Card 04 is the natural home.
