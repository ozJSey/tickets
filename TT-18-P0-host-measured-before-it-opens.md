# TT-18 — **P0.** The fit test measures the box the host had *before* it opened, and commits the verdict for life

Owner report, from a real app:

> "check in v-teleport-to, we are restricting max height or something, our tooltips have overflow:
> animated host, `data-teleport-state = open / closed` this only shows like 5 words."

**Reproduces against `dist` as well as `src`** (verdicts byte-identical across both).
Driven in real Chrome — jsdom cannot see this, for the reason in *Why nothing caught it*.

## Verdict: not TT-17, but TT-17 is a special case of it

TT-17 says the fit test never runs when `v-show` is written after the directive. The real defect is
one level down and has nothing to do with `v-show`:

> **`measureHostExtent` is asked for the host's extent at a moment when the host does not yet have
> the box it will have while open, and `resolvePlacement`'s answer is then written once and never
> revisited.**

`display: none` (TT-17) is the variant where that box is *absent*. The tooltip report is the variant
where it is merely *collapsed* — by the closed half of the consumer's own
`[data-teleport-state]` animation — and it is strictly worse:

| | TT-17 (`v-show` after the directive) | TT-18 (animated host) |
|---|---|---|
| rect at measure time | `0` → extent `null` | `18px` (padding+border) → extent `18` |
| `fit` reported | `'unmeasured'` — honest | **`'fits'` — a lie** |
| `data-teleport-collapsed` | absent | absent |
| how long it lasts | until a scroll/resize/re-render | **every open, for the life of the host** |
| caught by TT-17's acceptance ("both orderings agree")? | yes | **no** — both orderings are equally wrong |
| fixed by TT-17's proposed fix ("re-measure when the first measurement *fails*")? | yes | **no** — it does not fail |

**This is the sentence to carry into the fix:** TT-17's remedy is keyed on `measureHostExtent`
returning `null`. In the tooltip case it returns `18`. Ship TT-17 as written and the owner's bug is
still there.

A third variant, which is the one that should stop a publish: **the same truncation happens with no
animation at all**, on the first open of any host whose content re-wraps.

## Measured

Headless Chrome, viewport 1280×657. Geometry identical for every row:
trigger `position: fixed; top: 60px; width: 160px; height: 24px` →
`rawSpaces.top = 60`, `rawSpaces.bottom = 573`. Host = a 36-word sentence, `overflow: hidden`,
natural height **127.13px** (`scrollHeight` 125) at the 240px width the library itself writes
(`1.5 × 160`). `placement: 'top'`, default `maxHeight` (240), host **always mounted**, open/close by
`enabled` — i.e. playground card 10's recipe, which is where the owner's wording
("animated host", "`data-teleport-state = open / closed`") comes from verbatim.

### The owner's shape — a closed state that collapses the box

`[data-teleport-state="closed"] { max-height: 0 !important; opacity: 0 }` +
`transition: max-height 200ms`:

```
open #1  fit=fits  placement=top  max-height 60px   rect 60   scrollHeight 125
open #2  fit=fits  placement=top  max-height 60px   rect 60   scrollHeight 125
open #3  fit=fits  placement=top  max-height 60px   rect 60   scrollHeight 125
```
What the user can read, counted word by word against the host's content box:
```
13 of 36 words visible, 23 clipped — on all three opens
visible: "A tooltip that explains this control in a full sentence, which is what"
clipped: "tooltips are for, and which takes rather more than one single line of text
          to say properly at the width a tooltip gets."
```
`fit` says **`fits`**. `data-teleport-collapsed` is absent. Nothing on the public surface reports a
problem. The owner's "only shows like 5 words" is this, at his line width.

At measure time the closed rule still wins (the directive stamps `state="open"` at the *end* of
`calculatePosition`), so the rect is 18px = padding + border. 18 ≤ 60 → `'fits'` on `top` →
`max-height` = the 60px that side has. `height: 0 !important` and `transform: scaleY(0)` behave the
same way (`scaleY` reads rect `0` → `'unmeasured'`, same clamp).

### No animation at all — first open is wrong for any wrapping content

```
closed, never opened : position static, width 1280, rect 36.19   ← what the fit test will read
open #1              : fit=fits     placement=top     max-height 60px   rect 60    (13/36 words)
open #2              : fit=flipped  placement=bottom  max-height 240px  rect 127.13 (36/36 words)
```
The host is measured **before** the directive's own `position: fixed` + `width: 240px` are applied,
so it is measured while still in normal flow at the parent's full width, where the text needs two
lines (36.19px) instead of five (127.13px). 36.19 ≤ 60 → `'fits'` → clamped to 60px. Open #2 is
correct only because the inline `position`/`width` from open #1 are still on the element.

Pre-styling the host the way the library will anyway removes it completely:

```
consumer CSS `position: fixed; width: 240px`  → open #1 fit=flipped, max-height 240px, 36/36 words
```

### Why the menus never showed it, and a tooltip does

```
content = 5 block rows (a menu)  : closed rect 143   open #1 fit=flipped  max-height 240px  ✓
content = wrapping text (a tip)  : closed rect 36.19 open #1 fit=fits     max-height 60px   ✗
```
A menu's flow height equals its teleported height, so the stale measurement happens to be right.
**Text that re-wraps when the width changes is the only content shape where the pre-write box is
shorter than the final box** — which is exactly what a tooltip is.

`placement: 'auto'` also masks it (the comparative preference picks `bottom` before the lie can
matter: `fit=fits placement=bottom max-height 240px`). It takes an **explicit** placement — the
normal tooltip spelling — to expose it. A genuinely short tooltip is unaffected
(`rect 36.19`, `max-height 60px`, nothing clipped): there is no false positive, only a false "fits".

### Nothing re-measures the host, so it does not self-correct

`resolve-placement.ts:126` says a host measured mid-animation "under-reports — a transient that
self-corrects when the animation settles". **It does not.** The only recalc triggers are
`window` resize, `scroll` on the resolved targets, the `autoUpdate` observers — which
`auto-update.ts` attaches to the **reference**, never to the host — and a Vue re-render. A tooltip
on a page nobody scrolls never gets a second tick. Measured: three consecutive open/close cycles,
identical truncation; one synthetic `scroll` event fixes it instantly
(`fit=flipped placement=bottom max-height 240px`), which is the proof that the geometry was always
reachable and only the timing was wrong.

Two things *do* rescue it, and they explain why this was invisible in review: a scroll/resize tick,
and any re-render that happens **while the host is already open** — at that point the host's box is
our own 60px clamp, `measureHostExtent`'s see-through branch returns the consumer's `maxHeight`, and
the flip fires correctly. The bug needs the measure to land on a tick where the box is *smaller*
than the clamp we wrote (collapsed by the closed state) or where there is no clamp yet (first open).

## Why nothing caught it

- **Card 10 is the recipe the owner used** and cannot fail: `placement: 'auto'` (masks it),
  `maxHeight: 120`, and two short non-wrapping lines.
- Every fit-sensitive card (01, 02) is a **menu of block rows**, the one content shape whose flow
  height already equals its final height.
- **The test harness encodes the assumption that is false.** jsdom has no layout, so a host only has
  an extent if `sizeHost` stubs one, and `sizeHost`'s two modes are a fixed height or
  `min(natural, the max-height we last wrote)` (`vTeleportTo.test.ts:42`). Both *guarantee* the rect
  is at least as tall as the content allows. This bug is the case where the rect is **smaller than
  both** — collapsed by the consumer's closed-state CSS, or laid out at a different width before our
  own `width` is applied. No unit test can currently express it, so 806/806 green is not evidence
  here.
- `teleport-positioned` reports `fit: 'fits'`, so even a consumer who logs the detail is told
  everything is fine.

## Fix — and why I am not patching it here

Every honest fix changes **when** the host is measured, which is `measureHostExtent` /
`computePositionStyles` — the path both documented infinite-loop bugs came out of
(`scrollHeight` + an absolutely-positioned `::after` arrow; `offsetHeight`'s whole-pixel rounding).
The loop risk is not hypothetical in the animated case: with a `max-height` transition the host's
rect is a *function of the clamp this library just wrote*, which is precisely the
side → style → measurement → side cycle the header comments close by hand.

I measured that risk rather than assuming it. Forcing a recalc on every frame through a 400ms open
animation (45 frames, three animation shapes):

```
max-height 0→inline : 1 placement change (top→bottom at frame 11), then stable
scale(0.6)→scale(1) : 0 placement changes
scaleY(0)→scaleY(1) : 1 placement change (at frame 12), then stable
```
So per-frame re-measurement **converges** — but it converges by flipping the tooltip from above the
trigger to below it *halfway through its own open animation*, which is a visible jump the consumer
did not ask for. "Re-measure until it stops changing" trades a permanent truncation for a
mid-animation lurch. Choosing between those is a design decision for this library's owner, not a
patch to slip in, and it sits in the one function that must not be destabilised. Hence: ticket.

### Options, with what each must not break

1. **Re-measure once the host's box settles, not once per frame.** A `transitionend` /
   `animationend` listener on the host triggering a single `scheduleUpdate`. **Measured working:**
   the same collapsed-box variant with a consumer-side `transitionend → resize` nudge reports
   `fit=flipped placement=bottom max-height 240px` after the animation, on every open. Cheap
   (rAF-batched, one recalc per animation), and it cannot livelock: the recalc only writes styles
   that are not themselves transitioned in the common case. Does **not** fix the no-animation first
   open (no transition → no event), and fires a recalc for unrelated transitions (harmless, same
   cadence as a scroll tick).
2. **Measure after this tick's own layout writes, not before.** Two-pass: write
   `position`/`width`/`min-width` first, measure, then decide the side and write
   `max-height`/`top`/`bottom`. Fixes the first-open case at the source (it is the only reason the
   flow-width box is ever read) and is independent of animation. Breaks the
   "host's box is read exactly once per tick" invariant and turns one reflow into two — and that
   invariant exists because two reads let the fit test and the arrow math disagree about the host's
   width, which is T2 in TT-16. If this is taken, the arrow/cross-axis math must keep using the
   *same* rect it uses today.
3. **A `ResizeObserver` on the host** (under `autoUpdate`). This is the per-frame option above, with
   the classic RO-write-in-callback loop on top. Highest risk of the three; it is also the only one
   that handles a host whose content changes while open.
4. **Make the lie visible at minimum.** Whatever is chosen, a verdict taken from a host whose box is
   smaller than its own content must not be reported as `'fits'`. Distinguishing that state needs a
   content measurement the comments rule out (`scrollHeight`) — so it probably has to come from
   remembering that the extent was taken before the host was ever `state="open"`, and reporting
   `'unmeasured'`.
5. **Doc-only, shippable today, fixes nothing but stops the bleeding.** The README's
   `[data-teleport-state]` recipe currently teaches the animation and says nothing about the
   measurement. It must say: animate `opacity`/`transform` only; a closed state that collapses the
   box (`max-height: 0`, `height: 0`, `transform: scaleY(0)`) poisons the fit test for every open;
   and prefer `placement: 'auto'`. Also delete the false "self-corrects when the animation settles"
   claim in `resolve-placement.ts`.

### Must not break

- `scrollHeight` (an absolutely-positioned `::after` arrow inflates it) and `offsetHeight`
  (whole-pixel rounding) stay forbidden. *Note: the arrow variant in this repro measured
  `scrollHeight` 125, identical to the no-arrow variant, so the inflation did not appear in this
  shape — that is not evidence that it cannot, so do not reopen that door on the strength of it.*
- The `'auto'`-runs-the-fit-ladder fix (TT-15) and the two space records (buffered for preference,
  raw for fit) must survive untouched.
- `flip: false` must still take no measurement at all.
- `fit: 'fits'` must keep meaning "the host fits on this side".
- 806 tests stay green, and the fix must hold for **both** `v-show` orderings (TT-17) and for the
  composable's `flush: 'sync'` timing.

## Acceptance

- A real-browser regression that **fails first**, with an animated host whose closed state collapses
  the box, `placement: 'top'`, wrapping text taller than the cramped side: three consecutive opens
  must each report `fit: 'flipped'`, `placement: 'bottom'`, `max-height: 240px`, and **36 of 36
  words visible**. Word counting (or `scrollHeight` vs `clientHeight`) rather than "the attribute
  changed" — this is a bug whose every attribute currently reads healthy.
- A second regression for the no-animation first open: a host with wrapping text, no consumer
  `position`/`width`, must not be clamped to the cramped side on open #1.
- No mid-animation placement change in the fixed build: record `data-teleport-placement` on every
  frame of the open animation and assert it is constant (the per-frame numbers above are the
  baseline to beat).
- `sizeHost` gains a mode that can model a host **shorter** than `min(natural, clamp)` — a box
  collapsed by consumer CSS — otherwise the jsdom suite structurally cannot hold a regression for
  this, and the browser check is the only guard.
- Verify against `dist` — it reproduces there too, and `dist` is currently 16 minutes older than
  `src`, so rebuild before measuring.
- A playground card that is **not** a menu: wrapping text, explicit `placement`, a height-collapsing
  closed state. Card 10 cannot express this bug and should keep its `'auto'` binding; this needs its
  own card, or card 02 needs a wrapping-text content mode.
- TT-17 is closed by the same pass or explicitly re-scoped: its acceptance criteria do not cover
  this, and its proposed remedy does not fix it.

## Repro

Self-contained, no playground. Host always mounted, `enabled`-toggled, `placement: 'top'`,
trigger fixed at `top: 60px`:

```css
.tip { overflow: hidden; padding: 8px; border: 1px solid #345; }
.tip { transition: max-height 200ms, opacity 200ms; }
.tip[data-teleport-state="closed"] { max-height: 0 !important; opacity: 0; }
```
```
<div class="trig" ref="trigger">reference</div>
<div v-teleport-to="{ to: trigger, placement: 'top', enabled: open }" class="tip">
  36 words of text …
</div>
```
Open it, wait for the transition, then read back
`getComputedStyle(host).maxHeight`, `host.getBoundingClientRect().height`, `host.scrollHeight` and
`host.dataset.teleportFit`. Expect `60px / 60 / 125 / "fits"`.

The harness used for the numbers above (four pages: all animation shapes, mechanism isolation,
per-frame oscillation probe, word-visibility count) lives in this session's scratchpad at
`scratchpad/tooltip/` — `run.mjs` serves the directory, opens headless Chrome in real time and waits
for the page to POST its measurements back. Note for whoever rebuilds it: `--virtual-time-budget`
**deadlocks** on a page with CSS transitions, which is why the harness drives real time instead.
