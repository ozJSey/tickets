# SIV-2 — should the `container` path read CSS `scroll-margin-*`?

Deliberately deferred by the SIV-1 fix, with the reason recorded. Not a bug — an API trade.

The native path honours CSS `scroll-margin-*` because the browser does it. The `container` path
does not, because `getBoundingClientRect()` excludes scroll-margin, so the library never sees it.
That divergence is now documented in the README (`offset` is the mirror).

**Why it was not just implemented:** doing it natively means `end` alignment uses
`scroll-margin-bottom`, which contradicts the documented and tested meaning of `offset` — a blunt
subtraction applied on every alignment. Two different mental models for the same visual gap.

Decide one:
- **Read `scroll-margin` per side, native semantics**, and reconcile with `offset` — likely
  `offset` becomes an addition on top rather than the only mechanism. Breaking for anyone relying
  on the current `end` behaviour.
- **Keep the divergence**, and make the README section the canonical explanation (it already is).
- **Read it but apply it as `offset` does**, which is neither native nor the current behaviour and
  is probably the worst of the three.

Nothing is broken today; this is about whether the two paths should converge.
