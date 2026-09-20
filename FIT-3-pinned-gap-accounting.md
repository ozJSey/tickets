# FIT-3 — a pinned child's gap appears to go unbilled

**Status: QUARANTINED in CI, not fixed.** `pnpm geometry` carries
`continue-on-error: true` so the pipeline can reach green. Remove it the moment this is decided.

**`v-fit-children` is under a feature freeze** (BOARD #38, owner: *"We agreed, nothing on
vFitChildren"*), and the owner's words on it since: *"it's a really really surgical npm package… one
bad update away from infinite loop or break"*. So this ticket stops at diagnosis. The fix is the
owner's call.

## What is observed

Card `03-keep-visible`, published `@ozjsey/v-fit-children@2.3.3`, on Linux CI, deterministically,
at **all three** viewports (1400x1000, 1024x800, 375x812):

```
over-spill at w=240px
  visible span* 112px + div 105px + 1x6px gap = 223px against 219px available   (* = pinned)
clipped at w=240px
  host scrollWidth exceeds clientWidth by 4px
```

On macOS the identical card at the identical width reports `overflowPx -25, clipPx 0` — **25px of
free space**. Both measurements are honest: `system-ui` is a different typeface per platform, so the
chips really are wider on Linux. The defect only becomes visible when the numbers are tight.

Not a timing race: the three viewports and repeated runs produce byte-identical numbers. A race
would be flaky.

## The hypothesis the arithmetic points at

`fit.ts` reserves pinned children up front, **out of DOM order**:

> *"Pinned children are never hidden wherever they sit, so their cost is reserved up front rather
> than discovered part-way through the walk."*

`running` starts at the pin's cost, then each non-pinned child is admitted while
`running + cost[index] + metric.marginAfter <= budget + EPSILON`.

`cost` is built from each child's own width and its `spacingBefore`, and `spacingBefore` is
`rect.left - previousRight` — measured against the **previous child in DOM order**. When the pin is
reserved out of order, the gap between the pin and the first admitted child is billed against
whichever child physically preceded it, which may be one that ended up hidden. 112 + 105 = 217 fits
in 219; 112 + 6 + 105 = 223 does not. The overrun is 4px, the gap is 6px, `EPSILON` is 0.5.

**This is a hypothesis with arithmetic behind it, not a confirmed root cause.** It has not been
reproduced in a unit test, and macOS cannot reproduce it at all.

## What would settle it

A unit test in `v-fit-children` with a pinned child, a following child, a non-zero `gap`, and a
container sized so the two fit only if the gap is unbilled. It needs no browser — the fit
arithmetic is pure — and it either reproduces the 6px or refutes the hypothesis outright. Write
that before touching `fit.ts`.

## Why it was not fixed here

Every previous confident fix in this session that skipped the reproduce-first step made things
worse: `v-teleport-to@1.1.5` "fixed" a render loop by adding five self-dependencies and shipped the
loop to consumers. The scrollbar defect went the other way — diagnostic first, one-line proof, then
a fix that worked. This follows the second path.
