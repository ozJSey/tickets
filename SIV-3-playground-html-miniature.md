# SIV-3 — `v-scroll-into-view/playground.html` reimplements the directive by hand

Found during the SIV-1 fix. The package's standalone `playground.html` carries a **hand-written
miniature** of the directive — no `container`, no `offset`, and pre-1.2.0 `nearest` semantics —
whose comment claimed *"same logic as the package"*. It never was, and after 1.2.0 it is further
from the truth.

The comment was corrected to point at the real playground rather than rewriting the file, but it is
**dead weight that will drift again** — the same class as a stale README, and worse because it
looks like runnable proof.

Decide: delete it, or make it import the built artifact so it cannot drift. **Check the other
packages for the same pattern** — several have a `playground.html` and they predate the shared
playground app.
