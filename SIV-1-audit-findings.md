# SIV-1 — `v-scroll-into-view` audit findings

Independent audit, 2026-09-06. Verdict: **partly works — do not publish today.** Driven in a real
browser over CDP, through the playground (source *and* `dist`) **and** through a clean Vite consumer
app built outside the repo, so library defects could be told apart from playground defects.

The arithmetic is genuinely good — every alignment lands within ~1px. The defects are at the edges
the maths does not cover, and two are on the **default** `block: 'nearest'`.

## Library defects

**B1 — HIGH. A hidden target with `container` scrolls the pane to the top.**
| target | path | pane scrollTop |
|---|---|---|
| `v-show="false"` | `container` | 300 → **0** |
| `v-show="false"` | native (no container) | 300 → 300 ✓ |
| visible (control) | `container` | 300 → 240 ✓ |

`src/execute-scroll.ts:42-46` computes `relTop` from a `display:none` element's zero rect, giving a
large negative that clamps to 0. The code guards the **container** being detached
(`!container.isConnected`) but never the **target**. Reproduced in the playground *and* in the clean
app. jsdom cannot see it — every rect is zero there anyway, which is why 236 tests missed it.

**B2 — HIGH, default path.** `block: 'nearest'` + `container`, target taller than the container:
400px target in a 200px pane → library `scrollTop 477`, target top **-220** (shows its bottom).
Native `scrollIntoView({block:'nearest'})` → `scrollTop 256`, target top **+1** (shows its top).
The README claims **"full native API parity"** for `block`, and `nearest` is the default.

**B3 — MEDIUM, default path.** `nearest` + `offset` clips the target when its height equals the
container's `clientHeight`: 200px target in a 200px pane, `offset:{top:40}` → top 40 / bottom 240,
**40px cut off**, when top 0 was available. A 150px control is correct. Cause: the identity
heuristic at `src/execute-scroll.ts:69` — `if (newTop === relTop) newTop -= opts.offset.top` —
where `far - client` coincides with `rel` exactly when `size === client`.

## Documentation defects

**B4 — pairing with `focus()` silently voids the directive.** Instrumented, `container` + `nearest`
+ `offset:{top:56}`:
```
1-after-condition-flip  st=0    topInBox=584
2-after-focus()         st=500  topInBox=84    ← the browser did all of it
3-in-raf                st=500  topInBox=84    ← the directive did nothing
```
Without `focus()`: `st=417 topInBox=167`. The resting position is decided by Chrome, not by the
consumer's `block`/`offset`, and the two disagree by 83px. **The README mentions `focus()` nowhere**
— no `preventScroll`, no ordering guidance. *(Independently corroborates the KB-1 survey finding;
`v-keyboard-navigation` is being built against the correct behaviour.)*

**B5 — CSS `scroll-margin-*` is honoured natively and silently dropped with `container`.** Same
element, same `block:'start'`, `scroll-margin-top: 40px`: native → 40.5px gap; container → **-0.3px**.
A global `scroll-margin-top` for a sticky header is a very common setup that stops working the
moment `container` is added.

**B6 — default `behavior: 'smooth'` ignores `prefers-reduced-motion`.** With reduce emulated *and*
the pane computing `scroll-behavior: auto`, the scroll still animated across 24 distinct positions.
The explicit option beats the consumer's accessibility CSS reset. Undocumented.

## Tab and docs contradictions

- **Card 04 "Sticky-header offset" is completely dead** — `updated` fires **0 times**. This is
  **PG-14**, not the library; the identical template works in a real Vite app. It has almost
  certainly never worked in a browser.
- Card 04's blurb promises "with and without a container" — there is no container-less mode.
- Card 03's prose says the page scrolls "instead of the pane"; measured, **both** scroll
  (pane 0→882 *and* winY 0→562).
- The README's flagship `v-for` recipe **cannot scroll when pasted** — 5 items,
  `scrollHeight 200 === clientHeight 200` — and its `activeIndex++` walks off the end with no wrap.
- The README's `Cancel` button is disabled essentially always (`state` is `pending` for 1 frame in
  90 sampled). Card 07's prose says so; the README does not.
- **No card uses the bare binding or the container-less native path**, so the native path,
  `scroll-margin`, nested scrollers, already-visible targets and hidden targets are untested by the
  tab. Add coverage for the default configuration.

## Confirmed working — do not regress

Bare binding on mount (0→578, settles 545ms) · boolean edge detection incl. rewind-and-retrigger ·
all four `container` forms landing identically · nested scrollers (inner 0→388 centred at 50.1,
outer untouched) · already-visible target is a true no-op · malformed/detached/throwing containers
are silent no-ops with zero console errors · `start`/`center`/`end`/`nearest` × both axes × positive
and negative offsets, all within ~1px · sticky-header offsets 56.1/56.5/56.9/57.3 and left 80 ·
`scrollMarginTop` write-and-restore does not race the smooth animation · `always` chat autoscroll
settles at distanceFromBottom 0 in 227ms over 25 messages · `useScrollIntoView` · the plugin · the
`pending` CSS hook genuinely paints for one frame · `dist/` byte-identical to a fresh build, clean
tarball · 236/236 unit tests.

## Acceptance

- B1–B3 fixed with **browser** regressions; jsdom cannot express any of them.
- B4–B6 documented, or fixed where documenting is not enough (B6 arguably is a defect, not a doc gap).
- README's "full native API parity" claim either becomes true or is qualified.
- The tab gains bare-binding and native-path coverage.
- Blocked on **PG-14** for anything verified through a `v-for` card.
