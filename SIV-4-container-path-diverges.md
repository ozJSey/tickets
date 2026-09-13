# SIV-4 — the `container` path disagrees with native in three measured ways

Found by the **blind re-audit**, 2026-09-07, in the paths SIV-1 had just fixed. The container path
is the package's headline differentiator, and it is the part that diverges.

## S1 (high) — every scroll is off by the container's border width

Two identical panes, 24 rows x 40px, same `block`/`behavior`, no offset:
```
border=0   every block, every target:  native == container
border=10  block=start   target 0  ->  native   0   container  10
           block=start   target 5  ->  native 200   container 210
           block=center  target 5  ->  native 130   container 140
           block=end     target 5  ->  native  60   container  70
           block=nearest target 5  ->  native  60   container  70
```
`execute-scroll.ts` computes `relTop = targetRect.top - containerRect.top + container.scrollTop`.
`containerRect` is the **border box** while `scrollTop` and `clientHeight` are **padding-box**
relative, so `borderTopWidth` is never subtracted. Padding is handled correctly — verified with
`padding: 20px`, both paths agree exactly.

**Card 12 — the card built to prove native parity — reads `lib = nat + 1` on every row** because
`.pg-scroller` has `border-top: 1px`. Card 11 shows it too (686 vs 685).

## S2 (high) — `nearest` disagrees with native by a full pane height for an oversized target from below

Card 12, `size=400`, `from=below`:
```
directive:  scrollTop 229 · target top    0 · bottom 400   (near edge aligned)
native:     scrollTop 430 · target top -201 · bottom 199   (far edge aligned)
```
Independently: a 400px item in a 200px pane parked at 900 gives native 400, container 200.

CSSOM-View: when the element's **start** edge is before the scrollport **and** the element is taller
than the scrollport, the **end** edges align. `scrollFor`'s `startOutside && !endOutside` branch
always returns `rel - lead`. **The card's blurb, its prose and its `native parity` tag all assert
the opposite** — "taller than the pane aligns the top, not the bottom" — and its two readouts sit
201px apart on the exact case it is named after.

## S3 (medium) — `offset.top` applies to `center` and `end` on the container path only

```
block=end    offset none: native 320  container 320    offset 60: native 320  container 260
block=center offset none: native 400  container 400    offset 60: native 370  container 340
block=start / nearest:    the two agree
```
The native path implements the offset as `scroll-margin-top`, which CSS applies fully to `start`,
half to `center`, and **not at all** to `end`. The README documents exactly two deliberate
exceptions to "the alignments agree"; this is an undocumented third.

## Acceptance

- Browser regressions for all three, **each negative-controlled individually** — the SIV-1 pass set
  that standard and it held up.
- S1: a bordered container must agree with native for every `block` and every target.
- S2: match CSSOM-View, or change the card's claim and the README's parity list. **Do not leave a
  card asserting a parity its own readouts contradict.**
- S3: converge, or document it as a third exception.
- 272 tests stay green.
- **Note the re-audit's verdict:** the container-less path matched native everywhere it could be
  measured, and it would publish that alone. The divergence is entirely in the `container` feature.
