# TT-16 — `v-teleport-to` audit findings (beyond TT-15)

Independent audit, 2026-09-06. Verdict: **partly works — do not publish.** Four library defects
reachable with **no opt-in**, one broken README recipe, and five of twelve cards that cannot
demonstrate the feature they exist for. The core is good: bare-binding tracking, clip escape,
`hideWhenReferenceHidden`, `scrollContainer` + the window floor, virtual references, all the
`data-*` attributes, and `flip` itself all verified working. 764/764 unit tests pass.

**T1 — `matchWidth: true → false` welds `width` on forever.** `src/calculate-position.ts:331` sets
`styles.width` only when `matchWidth` is truthy, and `calculatePosition` writes only defined keys —
so nothing clears it. Every other side (`top`/`bottom`/`left`/`right`) is explicitly cleared with
`''` for exactly this reason; `width` was missed. Card 03, identical control state either side of
one on/off cycle:
```
initial (matchWidth false): renderedW 258.6  cssWidth (none)
matchWidth OFF again      : renderedW 220    cssWidth 220px   ← welded
after 5 forced recalcs    : renderedW 220    cssWidth 220px
```
Text clips to "maxHei…". Reproduces against `dist/`.

**T2 — `--teleport-arrow-x` is wrong whenever the host is right-anchored.** Default path: the
`anchorRight` heuristic fires automatically at 71% of viewport width, no option required.
`hostLeftViewport` is derived from `parentWidth × widthMultiplier` (the *projected* width) while the
host is positioned with CSS `right:`, so its real left edge is `parentRight − renderedWidth`.
Error = `projectedWidth − renderedWidth`.
```
trigger at 34% of vp, left-anchored  → --arrow-x 100px, err 0
trigger at 80% of vp, RIGHT-anchored → --arrow-x 200px, err 100  ← arrow on the host's corner
```
With `widthMultiplier: 3` the arrow lands **127px outside the popover**. `crossAxisAlign: 'end'`
reproduces it at any x. Reproduces against `dist/`.

**T3 — mobile full-bleed detaches every dropdown from its trigger.** Below 768px the directive
writes `left: 0` + `max-width: 100vw` but never `width`, so a host narrower than the viewport is
pinned to the screen's left edge while its trigger sits elsewhere. Card 01 at 400px, bare binding:
trigger at x≈300, menu at `left: 0`. `maxWidth` overrides the width half of this branch but not the
`left: 0` half — which the README's "overrides … the mobile fullbleed branch" does not describe.

**T4 — the README's `v-if` recipe is a one-way trapdoor.** Pasted verbatim:
```
ref on screen  → hostMounted true
ref off screen → hostMounted false   ← as advertised
ref back       → hostMounted false   ← never returns
```
`v-if` unmounts the host, which unmounts the directive, so no further `teleport-positioned` fires
and `gone` can never go back to `false`. If the trigger starts below the fold the recipe destroys
the host on its first tick.

**T5 — `maxWidth` / `widthMultiplier` cannot narrow the host below the reference.**
`minWidth: parentWidth` is written unconditionally, so `min-width` beats `max-width`. 220px
reference: `widthMultiplier: 0.5` → rendered **220px**; `maxWidth: 0` → rendered **220px**. The
README states *"`maxWidth: 0` is a valid value and collapses the host"* — it does not.

**T6 — composable + `strategy: 'absolute'` mispositions.** The README says it "falls back to
viewport-relative coordinates" — true of the numbers, but `position: absolute` resolves them against
the offsetParent, so the host lands **239px** from the trigger. The stated limitation reads benign;
the result is wrong placement.

**T7 — `dist/` freshness unproven.** Built 21:14; `src/calculate-position.ts` and
`src/scroll-target.ts` modified 21:46. Four behaviours matched across both modes, so not provably
stale — but rebuild before trusting. `CLAUDE.md` records this biting the repo three times.

## Cards that cannot demonstrate their own feature

- **05 `overflow`** — the strip is not scrollable in either axis (`clientWidth 935 == scrollWidth
  935`), and the reference sits nowhere near an edge. All 18 combinations produced byte-identical
  output. Its own instruction ("scroll the strip sideways") is impossible.
- **07 `crossAxisAlign`** — the rail is not scrollable, so "drag me right →" cannot move and the
  71% right-anchor heuristic the prose explains is unreachable. At `widthMultiplier: 1`,
  `legacy`/`start`/`center` are byte-identical.
- **08 `autoUpdate`** — both "Grow" buttons mutate Vue refs, so Vue calls `updated` and the host
  repositions **whether or not `autoUpdate` is checked**. The prose asserts the opposite. *(The
  library feature is fine — a direct DOM mutation outside Vue gives `autoUpdate=false → gap drifts
  −53.2px`, `true → gap 0`.)*
- **06 Arrow** — blurb says "drag the slider"; the card has one `<select>` and zero range inputs.
- **04 `boundary + scrollContainer`** — `onPositioned` is defined at line 34 and **never bound** to
  `@teleport-positioned`, so the status readout has always read `"—"`.

## Acceptance

- T1–T3, T5 fixed with browser regressions; all four reproduce in the built artifact, so verify
  against `dist` too.
- T4, T6 fixed or the recipes rewritten to something that works when pasted.
- The five cards demonstrate their features, or their prose stops claiming they do.
- Blocked on **PG-14** for anything verified through a `v-for` card.

## Coordination note

The audit reported the shared scratchpad root causing interference — a concurrent agent overwrote a
driver and its edits triggered Vite full-page reloads mid-run. **Give each agent its own scratchpad
subdirectory**, and do not run browser-driving agents against the same dev server concurrently.
