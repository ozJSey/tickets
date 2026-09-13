# DZ-3 — **P0.** Tabbing into a dropzone scrolls the page away from it

Found by the independent audit, 2026-09-06. **Bare binding, default path, reproduces against
`dist`, 10 of 12 cards affected.** `pnpm smoke` and `smoke:dist` are green with this live.

## Cause — and it is the CSS the DZ-1b brief specified

`src/constants.ts:54-57`, `PICKER_HIDDEN_STYLE`:
```
position:absolute; left:0; bottom:0; width:1px; height:1px; …
```
With a **static host** — the default, and exactly what the README quick-start CSS produces — the
containing block is the *initial containing block*, not the zone. So the picker input is positioned
at document `(0, ~viewportHeight)` regardless of where its zone is.

Measured with the README quick-start pasted verbatim, placed low on the page:
```
zone at document y 7087   ·   picker at document y 1312   ·   offsetParent = BODY
real Tab:  scrollY 6000 → 656 ,  activeElement = the picker ,  zone now 6431px below the fold
at 390px width the gap is 14,800px
```
Only demo 9 escapes, because it happens to set `position: relative`.

**This is the accessibility fix creating an accessibility bug.** DZ-1b made the picker focusable so
keyboard users could reach the feature; `clickToPick` then became default-on, so every consumer has
this tab stop. The README's Accessibility section prescribes `:focus-within` as the affordance — it
paints a focus ring on a zone the user can no longer see.

**The planner wrote that CSS into the brief.** The `left:0; bottom:0` anchoring was specified to
avoid creating a scrollbar, and the implementing agent verified "zero layout contribution", which is
true and is what it was asked for. Nobody asked where the input *is* relative to its zone.

## Fix

`position: relative` on the host is the fix. Decide how it is guaranteed rather than hoped for:

- **The directive sets it** when the computed position is `static` — reliable, but the directive
  starts writing layout to a host it does not own, which needs justifying and documenting.
- **Position the input relative to the host without a containing block** — e.g. `position: fixed`
  with coordinates, or `top`/`left` derived from the host rect, or `transform`. Weigh against the
  scrollbar problem the original anchoring was solving.
- **Document it and let the consumer do it** — weakest: it is a default-on feature and 11 of 12 of
  our own cards forgot.

Whatever is chosen must hold for a host that is static, one already `relative`, one inside a
transformed ancestor (which creates a containing block for `fixed` too), and one in a scrolling
container.

## Acceptance

- **Reproduce it as a failing browser check first**: bare binding low on a long page, real
  `Input.dispatchKeyEvent` Tab, assert `scrollY` does not move and the zone stays in view.
- Verify against **`dist`**, not just source — it reproduces there.
- Check all 12 cards afterwards; do not rely on demo 9's accidental `position: relative`.
- The README quick-start must produce a working zone **as pasted**, with no extra CSS the reader has
  to know about. If a rule is required, it is in the quick-start block, not only in a caveat.
- 309 unit tests stay green; add one that pins the containing-block requirement however it ends up
  being enforced.

## Also from this audit — file with it, same package

- **F2 (P1, library):** `autoUpload: false` + `upload` leaves `data-dropzone="active"` **forever**
  after a drop, and `DropzoneApi.state` reports `"active"` with it. `drag.ts:41-62` resets
  `dragDepth` but never calls `setState`; `process.ts:52-54` skips `setState` when `upload` is
  configured with accepted files, and the queue branch's comment claims it "keeps idle while
  pending" — nothing sets idle. A *pick* through the same card ends `idle`, so it is drop-specific.
  README recipe 8 verbatim reproduces it. **The public API is reporting a state that is false.**
- **F6 (P2, library):** `--dropzone-progress` / `--dropzone-files-pending` persist through the
  `rejected` state for the full `rejectDuration` after a settled batch (100/0 retained). The README
  says the vars clear on idle and that rejected drops do not touch them; `rejected` is an
  undocumented fourth persist state. A consumer who styles `[data-dropzone="rejected"] .bar` sees a
  full progress bar on a drop that uploaded nothing.
- **F7 (P2, default + docs):** `paste: true` alone is close to dead. `pasteOn` defaults to `'host'`,
  a `<div>` host is not focusable, and the only reliable focus target inside it is the picker input
  — so ⌘V does nothing until the user Tabs in. Clicking the zone opens a chooser and leaves
  `activeElement = BODY`. The Options table flags nothing on the `'host'` default row.
