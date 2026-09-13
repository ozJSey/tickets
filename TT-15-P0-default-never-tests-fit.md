# TT-15 — **P0.** The default configuration never tests whether the host fits

Owner, 2026-09-06: *"v-teleport-to is not ready. It sucks because I pulled that from actual
production code and by iteration we made it worse, basically this doesn't work, when teleported
item is cut we don't flip it. Demos show it but it doesn't work."*

**He is right, and this is a planning error, not an implementation error.** TT-7's agent built
exactly what its brief specified, with measured evidence and mutation testing. The brief specified
the wrong thing.

## Audit correction, 2026-09-06 — the diagnosis was right, the framing was wrong

**`flip` is not broken.** The independent audit drove it at every geometry: holds `bottom` down to
387.8px of room, flips at 337.8px, `fit` transitions `fits → flipped → neither` correctly.

**Flipping is unreachable from the default**, and the failure mode is worse than "does not flip" —
the library clamps `max-height` to **zero**:
```
{"placement":"bottom","fit":"unmeasured","maxHeight":"0px","renderedH":10,
 "contentH":366,"visibility":"visible","roomBelowRef":127.7,"roomAboveRef":659.7}
```
A 366px menu rendered as a 10px empty box with 660px of unused room above it. On card 04 it is a
21px sliver with the title sliced through the glyphs. **`visibility` stays `visible`** — there is no
signal at all, and the README's own diagnostic for this dead end (`fit: "neither"`) can never fire
on the default placement.

**Two amplifiers, both new:**
- `BUFFER_BOTTOM = 150` / `BUFFER_TOP = 50` (`src/constants.ts`) are **absolute constants applied to
  whatever `boundary` is passed**, so a 200px scroll pane loses 150 of its 200px to a buffer sized
  for a viewport. Not configurable. Any fit-aware `'auto'` must reckon with this or it will conclude
  "neither fits" constantly in small boundaries.
- The zero-clamp is silent. Whatever the fit logic decides, **a host clamped to zero must surface
  it** — that is TT-13, and it is now the difference between a bug and an invisible bug.

## The mechanism

`src/resolve-placement.ts`:
```js
if (placement === 'auto') {                                    // ← the DEFAULT
  return { side: spaces.bottom >= spaces.top ? 'bottom' : 'top', fit: 'unmeasured' }
}
if (!flip) return { side: placement, fit: 'unmeasured' }       // ← flip is opt-in
```

1. `placement: 'auto'` is the default and **returns before the fit test runs.** It picks the
   roomier side comparatively and never asks whether the host fits there.
2. `flip` still defaults to `false`, so the other escape hatch is off too.
3. `maxHeight` then clamps the host into whatever space that side has — so a host that does not
   fit is **squashed or cut rather than moved.**

Net: on a bare `v-teleport-to="{ to: ref }"`, nothing in the library ever asks whether the thing
fits where it was put. **9 of the 12 demo cards use that default**, which is why the tab looks like
it demonstrates flipping while not doing it.

The TT-7 brief said: *"`placement: 'auto'` stays vertical-only and comparative — it is documented
that way and is a different feature."* That sentence is the bug. The README described the old
behaviour and was treated as ground truth rather than as another thing that might be wrong.

## What to build

**The bare binding must put the popover where it fits.** Consistent with the three other defaults
the owner has flipped this week (teleport hide-when-hidden, dropzone click-to-pick, v-copy dedupe)
and with `tickets/_STANDARDS.md`: *the default is what you would have chosen; options exist to opt out.*

1. **`'auto'` becomes fit-aware.** It should pick a side the host fits on, and fall back to the
   comparative rule only when neither side can hold it. Reuse the existing machinery — the fit
   test, `measureHostExtent`, and the `fits | flipped | neither | unmeasured` verdict are all sound
   and stay.
2. **`flip` defaults to `true`.** An explicit side becomes a *preference* honoured while it works,
   not a sticky instruction that lets the host be cut. Keep `flip: false` as the opt-out for
   consumers who genuinely want a pinned side.
3. **Interrogate the `maxHeight` clamp, which is the other half of the failure.** Clamping is what
   converts "does not fit" into "silently cut" instead of into a flip. Decide and document: does
   the clamp apply only *after* placement has chosen the best side it can? Is `fit: 'neither'`
   the only case where a clamped host is acceptable? Today a consumer sees a squashed popover with
   no signal unless they read `data-teleport-fit`.

## Do not regress what is already proven

`resolve-placement.ts` carries hard-won measurements — read its header comments before touching it:
- `getBoundingClientRect()`, not `offsetHeight`: whole-pixel rounding made a 39.531px host
  indistinguishable from one clamped at 40px, and the host flipped forever.
- Not `scrollHeight`: it counts the absolutely-positioned `::after` arrow, so the host measured
  ~5px taller on one side and the two sides took turns forever.
- `measureHostExtent` returns the *requested* `maxHeight` when the applied clamp is binding, so a
  clamped host cannot claim it fits. **This guard is what makes fit-based flip work at all** —
  keep it and make sure the `'auto'` path uses it.
- `CLAMP_EPSILON` exists because CSSOM re-serializes and Blink quantizes to 1/64.

Both of those livelocks were found only by driving a real browser. Any change here can reintroduce
them.

## Acceptance

- **Reproduce the owner's complaint as a failing test first**, in a real browser: a default-bound
  popover whose reference sits near a viewport edge, where the host is cut. Then make it flip.
- The full suite green — baseline **764** across 5 projects — with the `'auto'` tests audited: many
  currently assert the comparative behaviour that is being replaced, so they are not evidence.
- **Mutation-test:** reverting `'auto'` to comparative-only must fail tests; reverting the `flip`
  default must fail tests.
- **Browser-verify on the real cards, not a fixture.** Nine cards use the default; a bare dropdown
  near the bottom of the viewport must flip above rather than be cut. Check `01-basic-dropdown`,
  `09-virtual-reference` and `11-composable` specifically.
- README and `types.ts` jsdoc updated: `auto` and `flip` both change meaning. The manifest blurb
  for `02-placement-flip` and the card's prose need re-reading against the new default.
- Re-check the composable path — `useTeleportTo` never sees the host and reports `'unmeasured'`, so
  it degrades to comparative. Say plainly whether the default is therefore weaker there, rather
  than letting it be discovered.

## Planner note

Two failures produced this, both mine, and both worth avoiding on the next ticket:
1. **I treated the README as the specification.** It documented `auto` as comparative, so I froze
   that behaviour into the brief instead of asking whether it was right.
2. **I never asked what the bare binding does.** Every ticket this week asked "does this option
   work"; none asked "what happens with no options at all", which is the configuration almost every
   consumer actually ships and 9 of 12 cards demonstrate.
