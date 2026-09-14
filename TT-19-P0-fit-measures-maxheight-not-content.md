# TT-19 — **P0.** Flip should maximise VISIBILITY; it currently compares against `maxHeight`

> **CLOSED in `v-teleport-to` 1.1.0 (2026-09-14).** TT-17, TT-18 and TT-19 were
> one defect and were fixed as one: `src/measure-host.ts` now takes the host's
> extent as a single synchronous probe with the closed state, an inline
> `display: none`, our own `max-height` clamp and the side coordinate all lifted
> for the duration of the read, and the whole `style` attribute restored
> verbatim afterwards. Verified in Chrome against `src` and `dist`:
> `playground/scripts/interactions/v-teleport-to.mjs`, seven checks over cards 02,
> 13 and 14, six of which fail against the pre-fix measurement — one of them
> reporting the original symptom exactly, *13 of 36 words shown with
> `fit: 'fits'`*. 836 unit tests (was 806), eight mutations of the fix each turn
> the suite red — including one per forbidden measurement source. Both livelocks re-checked, not assumed: an arrow-bearing host
> reports the same `contentHeight` on both sides while its `scrollHeight`
> differs by 12px, and a host with a fractional natural height under a
> whole-pixel clamp is stable across 60 forced recalcs.

## The goal, in the owner's words (2026-09-13, from the live site)

> "v-teleport-to needs to respect visibility better, isn't flip supposed to enable FLIP if it's
> going to be visible at top instead of bottom?"

**That is a better specification than the one this ticket was written against.** "Flip when it does
not fit" was a proxy for "flip so more of it is visible", and the proxy is what is broken.

The rule the library should implement, stated as an ordered ladder:
1. The whole host is visible on the preferred side → **stay**.
2. It is not, and the whole host would be visible on the opposite side → **flip**.
3. Neither side shows all of it → **show the most of it**, and say so (`fit: 'neither'`,
   `data-teleport-collapsed`).

Rule 3 is already the comparative fallback; it just needs to be justified by visibility rather than
by "there is no right answer". Rules 1 and 2 are the ones the current measurement gets wrong.

**Worked example of the failure, which is exactly what the owner saw:** 300px of content, 250px
below the reference, 500px above. `DEFAULT_MAX_HEIGHT` is 240, so `measureHostExtent` reports the
host wants **240**. 250 ≥ 240, so the preferred side "fits" — the host stays below, is clamped to
250, and **loses 50px of content while the side above would have shown all 300.** `fit` reports
`'fits'`. Nothing signals a problem.

**Everything else on the tab is confirmed good by the owner** — he tested the live site and this is
his only finding. So the placement maths, the hide-when-hidden behaviour, the arrow, the boundary
and scroll-container handling are all working. This is the one thing left.

# Original ticket — the mechanism

Raised by the owner from production use, 2026-09-13, and confirmed in the source:
> "Do we rely on user setting the height in order to understand we need to flip?"

**Yes. And the content's real size never enters the decision at all.**

## The two lines

`src/resolve-placement.ts`, `measureHostExtent`:
```js
const applied = Number.parseFloat(hostEl.style.maxHeight)
if (Number.isFinite(applied) && rect.height >= applied - CLAMP_EPSILON) return maxHeight
return rect.height > 0 ? Math.min(rect.height, maxHeight) : null
```

- **Clamped branch** — the case where a flip decision actually matters — answers with the
  `maxHeight` **option**, not the content.
- **Unclamped branch** is `Math.min(rect.height, maxHeight)`, so the measured extent **can never
  exceed `maxHeight`** by construction.

`DEFAULT_MAX_HEIGHT = 240` (`src/constants.ts:34`). So on a bare binding the library believes every
host wants at most 240px, any side with 250px of room always "fits", and **content taller than 240px
is clamped and cut while `fit` reports `'fits'`** — no flip, no signal.

## The reported symptom

A tooltip with more than `maxHeight` of content renders about five words. The library never
considered flipping, because as far as it knows the content already fit. The owner's report:
*"we are restricting max height or something, our tooltips have overflow … this only shows like 5
words."*

## Why this is hard, and what not to repeat

Every obvious way to read the content's natural height has already been tried and each failed in a
real browser. **Read the header comments in `src/resolve-placement.ts` before proposing anything:**

- **`scrollHeight`** counts absolutely-positioned descendants — including the `::after` arrow pinned
  to whichever edge faces the reference. The host measured ~5px taller on one side than the other
  and the two sides **took turns forever.**
- **`offsetHeight`** rounds to whole pixels, so a host naturally 39.531px tall is indistinguishable
  from one clamped at 40px. **Second infinite loop.**
- **`getBoundingClientRect()`** returns the *clamped* height, which is circular: you cannot learn
  what the clamp is hiding by measuring through it. That circularity is exactly what the
  `maxHeight` fallback was papering over.

## Directions worth evaluating — none is obviously right

1. **Measure unclamped, once.** Remove the `max-height` (or set it to `none`), read the height,
   restore, in one synchronous block before paint. Costs a forced layout per tick; must not produce
   a visible flash; and the arrow problem returns if `scrollHeight` is used to read it, so read the
   rect with the clamp lifted instead.
2. **Measure the content, not the host** — sum the children's boxes, or use a wrapper whose height
   is the content's. Excludes absolutely-positioned children naturally, which is what killed
   `scrollHeight`. Changes what the host may contain.
3. **Let `max-height` stop being the mechanism.** If overflow were handled by scrolling the host
   rather than clamping it, the host's natural height stays readable. Much larger change.
4. **Signal it instead of solving it.** If the content does not fit on *either* side, say so —
   `data-teleport-collapsed` exists but cannot fire here, because the library thinks it fits.
   Weakest option, but it turns a silent truncation into a visible one.

## Acceptance

- **A regression that fails first**, with content meaningfully taller than `maxHeight`: a
  400px-content tooltip near the bottom of the viewport must either flip to a side that holds it,
  or report that neither side does. It must not silently clamp and report `'fits'`.
- Both historical livelocks must be re-checked explicitly, not assumed absent: an arrow-bearing
  host, and a host whose natural height lands on a fractional pixel. **Mutation-test both.**
- **Card 02 cannot catch this bug and needs a second axis.** It varies the *reference* height, which
  exercises available space; it never varies *content* height, which is the input this defect lives
  in. Add that control, or add a card for it.
- Verify against `dist`.
- 806 tests stay green.

## Related

**TT-17** (P0) is the sibling: when `v-show` is written after the directive — the README's own order
— the host is `display: none` at measure time and `fit` stays `'unmeasured'` permanently. Both are
the same family: **the fit test trusting a number that does not describe the content.** They should
probably be solved together, and a fix for one that ignores the other is half a fix.
