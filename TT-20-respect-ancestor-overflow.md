# TT-20 — the host must respect an ancestor's `overflow`, and pin rather than detach

Owner, 2026-09-13, from the deployed site:
> "also that last fixed example is visible when container is fully scrolled. We can pin it to top or
> bottom based on element position but never detached teleport-to. It needs to respect parent
> container overflow."

Two asks, and the second is the interesting one.

## 1. Ancestor clipping — the case TT-3 explicitly declined

`DESIGNS.md` → TT-3 states it outright: *"A reference clipped by an intermediate `overflow: hidden`
ancestor rather than by the viewport is **not** caught, and that is deliberate."* The reasoning was
that `getBoundingClientRect()` reports the layout box regardless of ancestor clipping, so the
reference's rect looks on-screen; and that walking the clipping-ancestor chain the way Floating UI
does would cost the library's stated core invariant — *reads only the reference's rect, never an
ancestor's overflow or clip styles* (`README.md:4`) — plus `getComputedStyle` per ancestor per RAF
tick.

The escape hatch offered was `boundary: <the pane>`. **The owner has hit this in real use anyway**,
so the default is wrong even if the reasoning was sound.

**There is a nearly-free route the same design already identified and shelved.** `auto-update.ts:133`
constructs an `IntersectionObserver` — `new IOCtor(() => trigger(), { root })` — and **throws its
entries away**. Those entries carry `isIntersecting` and `intersectionRatio`, computed by the browser
with **full ancestor-clip accuracy**, for free, on an observer that already exists and already fires.
TT-3 called this "the most valuable unclaimed signal in the codebase" and deferred it only because
it would make one option behave with two different accuracies depending on whether `autoUpdate` is
on.

That objection is weaker than the bug. Options, to weigh and choose:
- **Use the IO entries when available**, and document that ancestor-accurate clipping requires
  `autoUpdate` — the two-accuracies problem, made explicit rather than hidden.
- **Always construct the IO** for visibility, independent of `autoUpdate`. Costs one observer per
  host; buys one consistent answer.
- Walk the clipping ancestors. Most accurate, and the one that breaks the stated invariant.

## 2. Pin to the container's edge — never detach

*"We can pin it to top or bottom based on element position but never detached teleport-to."*

This is a **different behaviour from hiding**, and better for the case he is describing. When the
reference scrolls out of its container, rather than the host either floating free at a stale
position or vanishing, it **clamps to the container's edge on the side the reference left** — so it
stays visually attached to the thing it belongs to.

Decide how this composes with `hideWhenReferenceHidden`, which is currently on by default and hides
outright:
- Is pinning a third mode (`shift`-like) alongside hide?
- Is pinning what should happen at the *container* boundary while hiding is what happens at the
  *viewport* boundary?
- Does pinning have a limit — a reference 3000px above the pane should presumably hide eventually
  rather than sit pinned forever.

`overflow: 'shift'` already clamps along the placement axis, so there may be an existing mechanism
to extend rather than a new one to invent.

## Identify the card first

"That last fixed example" is ambiguous — confirm which card on the deployed tab he means before
assuming. `04-boundary-scroll` and `12-strategy-absolute` are the likely candidates, and the fix
must hold for both.

## Acceptance

- A reference scrolled out of a **scrolling ancestor** — not the viewport — is respected on the
  **default** binding, with a browser regression that fails first.
- The host pins to the container edge rather than detaching, and the composition with
  `hideWhenReferenceHidden` is documented, not incidental.
- Whatever is chosen, `README.md:4`'s "reads only the reference's rect" claim is either kept true or
  corrected — it is a stated invariant and it must not quietly become false.
- Nested scrollers, a transformed ancestor (which creates a containing block), and a pane inside a
  pane.

## Measured, 2026-09-14 — reproduced on the default binding, both strategies

Confirmed while closing TT-17/18/19; **not fixed there**, see the scoping note
below. Headless Chrome, playground card **`12-strategy-absolute.vue`** — which
is "that last fixed example": it is the last card on the tab, its `strategy`
select offers `fixed`, and it is the only card whose reference lives *inside* a
scroll pane with no `boundary` set.

Pane scrolled to the bottom so the reference has left it, in both strategies:

```
pane            y 567 … 747
reference       y 403          → 164px ABOVE the pane's top edge, clipped, invisible
refInsidePane   false
refInsideViewport true          ← this is the whole bug
host            visibility: visible, data-teleport-hidden absent, y 436
hostOverlapsPane false          ← floating over the page, outside the pane entirely
```

`hideWhenReferenceHidden` is on and does not fire, correctly by its own
definition: it tests the reference against `intersect(boundary, viewport)`, and
`boundary` defaults to `'viewport'`. `getBoundingClientRect()` reports the
reference's layout box whatever an ancestor's `overflow` does with it, so the
reference reads as on-screen. Both `strategy: 'fixed'` and
`strategy: 'absolute'` behave identically — this is not a strategy bug.

The documented escape hatch does work: `boundary: <the pane>` makes `clipRect`
the pane, the reference reads as fully outside it, and the host hides. That is
what card `04-boundary-scroll.vue` sets, and it is why card 04 looks correct
while card 12 does not. **The default is the bug.**

## Scoping — deliberately NOT bundled into 1.1.0

1.1.0 closed TT-17/18/19, which were one defect in `measure-host.ts` (the host's
extent). This is a different defect in a different place, and the two share no
code:

| | TT-17/18/19 | TT-20 |
|---|---|---|
| code path | `measure-host.ts`, the fit ladder, `maxHeight` | the reference-visibility predicate, `boundary` resolution, `auto-update.ts`'s IO |
| what changes | a measurement that was already meant to describe the content | a **default-on behaviour**, and on two of the three options a **stated core invariant** (`README.md:4`) |
| risk if wrong | a popover on the wrong side | a popover that vanishes when the consumer wanted it visible |
| test matrix | collapsed boxes, re-wrapping text, directive ordering, two livelocks | nested scrollers, transformed ancestors, a pane inside a pane |
| needs a decision from the owner | no — the ladder was already specified | **yes** — is pinning a third mode or a second boundary, and where is its limit |

Shipping them together would mean the measurement fix — the thing actually
being waited on — could not be released without also releasing a default-hiding
change, and could not be rolled back separately if that change turned out wrong
for someone. Kept apart on purpose.
