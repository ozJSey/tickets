# Quality audit — `v-teleport-to`

**Verdict: structurally-unsound**  ·  13 findings


## The single worst thing

The one decision this library exists to make — which side the host goes on — is fed by the library's own previous output, on BOTH axes, and the module that owns the decision spends 60 lines of header prose asserting the opposite. Vertically, `measureHostExtent` reads a box it clamped with its own `max-height` and papers over the circularity by substituting the consumer's `maxHeight` option (already ticketed as TT-19). Horizontally there is no paper: the host is positioned with a CSS `left`/`right` coordinate and `width: auto`, so the browser shrink-to-fits the host into the very side being tested, and `rawSpaces[side] >= hostExtent` becomes `room >= min(natural, room)` — true by construction. Measured in headless Chrome with the library's exact writes: room 240px, natural width 320px, rendered width 240px, verdict `fits`, 1000px of unused room on the other side. `resolve-placement.ts:141` says of that axis: "there is no loop to close." The fit test can only ever flip a horizontal popover by accident — because the measurement lags a tick and the room changed since — so whether `placement: 'right'` flips depends on the user's scroll history rather than on whether the host fits. Two P0 tickets (TT-18, TT-19) already circle the vertical half of this; nobody has noticed the horizontal half, because the jsdom harness stubs a constant host width that no browser would ever report.


## Findings

### 1. Horizontal fit test is a tautology: it compares the room against a width the library derived from that room  ·  `rot`

**src/resolve-placement.ts:165**

For `placement: 'left'|'right'` the host gets a CSS `left` (or `right`) coordinate and `width: ''` (calculate-position.ts:411, unless matchWidth), so per CSS 10.3.7 the used width is shrink-to-fit capped at `containingBlock − left` — i.e. capped at the space on the chosen side. `measureHostExtent(el,'x')` returns that rect width, so `resolvePlacement` evaluates `room >= min(natural, room)`, which is always true once the browser has squeezed the host. Measured in headless Chrome replaying the library's exact writes (viewport 1280, 40px reference at x=1000 → roomRight 240, roomLeft 1000, maxWidth 320): natural 320px, rendered 240px == roomRight, verdict `fits`, host wrapped to 54px tall with 1000px free on the left. A `white-space: nowrap` control rendered 257.7px and DID flip — so the test only functions for content that cannot be squeezed, or when `min-width` (= the reference's width) exceeds the room. The vertical axis has an explicit counter-measure for exactly this circularity two lines below (the `style.maxHeight` see-through branch); the header comment at :141 declares the x axis needs none because 'the width this library writes does not depend on which side was chosen' — it is the left/right coordinate, not the width option, that does the squeezing.

*Symptom:* A left/right popover stays pinned to the cramped side, wrapping into a narrow column, while `fit`, `data-teleport-fit` and the `teleport-positioned` detail all report `fits` and `availableSpace` looks healthy. Flipping it 'works' in some sessions and not others, because the only thing that ever breaks the tautology is the measurement lagging a tick behind a room that just changed.

### 2. Horizontal crossAxisAlign aligns the host from the maxHeight OPTION while the measured rect sits in scope  ·  `rot`

**src/calculate-position.ts:428**

`crossAxisAlign: 'center'` computes `hostTop = refCentreY − maxHeightStyle/2` and `'end'` computes `hostTop = refBottom − maxHeightStyle`, where `maxHeightStyle` is `min(room, maxHeight option)` — default 240. The host's real height was read into `hostRect` at line 248 and is never consulted. Confirmed with a probe against the real directive: a 40px-tall host with default `maxHeight` and `placement: 'right', crossAxisAlign: 'center'` gets `top: 0px` where centred is `top: 100px`, and `--teleport-arrow-y: 120px` — the arrow variable points 80px BELOW the bottom edge of a 40px host. `'end'` gives `top: -100px`, i.e. a 'bottom-aligned' host placed above the top of the viewport. This is the same defect class as TT-19 one layer over: a decision fed by a config value when the real measurement is already in hand. It survives because every unit test for it (vTeleportTo.test.ts:3145, 3156, 3174, 3182, 3233) runs an unsized 0-height host and derives its expected number from `maxHeight/2` — the assertion is the formula, so it can only agree with the code.

*Symptom:* A tooltip asked to centre beside its trigger renders one `maxHeight/2 − hostHeight/2` gap too high (100px at the defaults), and its arrow is pinned outside the popover entirely. Shrinking `maxHeight` 'fixes' it, which is what the README tells people to do, so consumers learn to tune a height option to correct a vertical offset.

### 3. crossAxisAlign 'center' IS a width feedback loop; the comment three lines above says it cannot be  ·  `flake`

**src/calculate-position.ts:341**

The comment at :339 reads 'Safe against the feedback loop that governs the height axis: nothing here makes the host's width depend on the side chosen or on its own left edge.' But `hostWidthActual` is the rendered width, line 473 writes `left = refCentre − hostWidthActual/2`, and for a `position: fixed` box with `left` set and `width: auto` the browser caps the width at `viewport − left`. So the left coordinate constrains the width, and the width determines the next left. Measured in headless Chrome, reference centred at x=1200 in a 1280 viewport, maxWidth 320: width 320 → 240 → 200 → 180 → 170 → 165 → 162.5 → 161.25, converging on 2×(W−centre)=160, with `--teleport-arrow-x` trailing one tick behind the whole way (160 emitted while the host was 240 wide — off-centre by 40px). `hostLeftViewport` at :528 repeats the same derivation, so the arrow inherits it. Each step needs only one tick source (a scroll frame, a resize, an `autoUpdate` observer), and with `autoUpdateSubtree` on a reference that contains the host the loop feeds itself, because each tick writes a genuinely different `left`.

*Symptom:* A centred popover near the right edge of the window gets narrower every time anything ticks, and drifts off-centre from its trigger while doing it — its final width depends on how many frames fired, not on the geometry. The arrow is off-centre by half the per-tick shrink for the whole sequence.

### 4. useTeleportTo leaves half its public output stale when the reference goes away, including a hidden `styles` reported as not hidden  ·  `rot`

**src/use-teleport-to.ts:182**

The `!result` branch resets `collapsed`, `referenceHidden`, `hidden`, `state` and `prevPlacement` and returns — it never touches `styles`, `placement`, `fit`, `availableSpace`, `oppositeSpace` or `maxHeight`. Probed against the real composable: with the reference below the fold, `styles.visibility === 'hidden'` and `hidden === true`; set `to = null` and `hidden` becomes `false` while `styles.value.visibility` is still `'hidden'` — the consumer binds that record with `:style`, so the host is invisible and the composable's own flag says it is not hiding it. The branch's comment claims precisely the opposite ('the composable reports the same "not hidden by us" answer rather than a stale true'). `placement` stayed `'top'` where its JSDoc at :55 promises `null` 'if the reference element is missing/detached', and `fit` keeps its last verdict where :68 promises `'unmeasured'`.

*Symptom:* A popover driven by the composable goes permanently invisible when its trigger unmounts, with every diagnostic ref saying it is fine; `:data-teleport-placement="placement"` keeps a side that no longer exists, so arrow-rotation CSS locks onto the wrong edge until the next successful tick.

### 5. The flip ping-pongs forever when the host's natural extent equals the room on the preferred side  ·  `flake`

**src/resolve-placement.ts:170**

`measureHostExtent` infers 'our clamp is binding' from `rect.height >= applied − 0.0625` and answers that case with the `maxHeight` option instead of the measured height — so the decision's input jumps discontinuously at exactly the point where the decision turns over. A host whose content is exactly as tall as the room is indistinguishable from one truncated by the clamp. Probed through the real directive (viewport 800, reference at 620..660 → room below 140, host natural 140, default maxHeight): placements across six ticks were bottom, top, bottom, top, bottom, top, with `onPlacementChange` firing all six times. The band is narrow (|natural − room| < 1/16px) but it is not exotic: integral row heights and integral boundaries hit it exactly, and a reference in a sticky header keeps `room` constant, so every scroll event anywhere on the page produces another flip. The suite has a sub-pixel test (:1192) and a 'neither survives the clamp' test (:1149) but nothing on the equality case.

*Symptom:* A dropdown that visibly vibrates above/below its trigger for the whole duration of a scroll, re-triggering any `onPlacementChange` animation on every frame — and a bug report that cannot be reproduced unless the menu happens to have exactly that many rows.

### 6. data-teleport-collapsed is the one state attribute the dormant path forgets to clear  ·  `rot`

**src/directive.ts:103**

The `updated` dormant branch (no `to`, or `enabled: false`) clears `data-teleport-placement`, `data-teleport-fit`, both arrow custom properties, releases the hide and writes `data-teleport-state="closed"` — but never `delete el.dataset.teleportCollapsed`, which `calculatePosition`'s own null branch at :766 does delete. Probed: a collapsed host (max-height 0) given `enabled: false` keeps `data-teleport-collapsed` while everything around it is correctly cleared. README.md:211 states these attributes are 'cleared when the directive is disabled (enabled: false), or when its to reference becomes missing or detached', and the README's suggested CSS for this very attribute is `display: none`.

*Symptom:* Paste the README's own `[data-teleport-collapsed] { display: none }` snippet and a host that was ever collapsed stays `display: none` for the whole time the directive is dormant, with no attribute anywhere to explain why a dropdown stopped appearing.

### 7. Three public JSDoc blocks in types.ts document behaviour the code replaced  ·  `lie`

**src/types.ts:181**

types.ts is the IDE surface — it is what a consumer actually reads — and it is out of date in three places where the README and the code agree with each other against it. (1) :181 `flip`: 'the placement degrades to the comparative rule (pick the side with more room, ties going to the preferred side)' when the host cannot be measured. resolve-placement.ts:220 returns the PREFERRED side and its own header explains at length why degrading would be wrong. Same error restated at use-teleport-to.ts:68. (2) :222 `strategy`: 'The useTeleportTo composable does not know the host element it will be applied to; in the composable, "absolute" always uses the viewport fallback. Consumers who need true offsetParent-relative coordinates should reach for the directive form.' The composable has taken a `host` argument since the TT-16 T6 fix and resolves `offsetParent` from it. (3) :435 `scrollContainer`: 'events elsewhere in the document do not trigger recalcs' — false by default, because scroll-target.ts:63 unions `window` in whenever `hideWhenReferenceHidden` is on, which is the default. The README row for the same option documents the union correctly.

*Symptom:* A consumer reads the type hint, believes `placement: 'bottom'` silently becomes 'top' on a host with no laid-out box, and either writes a defensive workaround or avoids the composable for absolute positioning — and a consumer scoping `scrollContainer` for performance gets two listeners where the docs promised one narrow one.

### 8. ARCHITECTURE.md's dependency table is wrong, and its test-harness note documents the old placement rule  ·  `lie`

**ARCHITECTURE.md:16**

The document's stated purpose is 'a strictly downward dependency graph' with 'No upward edges' (:45), and the auto-update row lists its dependencies as `types`, `vue (toValue)`. auto-update.ts:15 imports `isVirtualReference` from `./calculate-position`. The ASCII graph below omits auto-update and scroll-target entirely, so the one artefact that is supposed to hold the module boundaries honest does not list the edges it is asserting about. Separately, 'Adding a feature' item 5 (:79) tells the next author that a test which forgets `sizeHost` is 'silently exercising the comparative fallback instead' — that stopped being true when the unmeasured branch started keeping the preferred side, which this same document states correctly two screens up in the resolve-placement row. Nothing (no lint rule, no test) checks any of it.

*Symptom:* The next person to split or move a module trusts the table, moves `isVirtualReference`, and closes an import cycle the document claims is impossible; the next person to write a fit test trusts item 5 and believes an unsized host exercises a fallback that no longer exists.

### 9. `updated` subscribes to scroll where `mounted` deliberately refuses, and the RAF path has no missing-`to` guard  ·  `rot`

**src/directive.ts:93**

mounted:38 is explicitly `opts.to ? resolveScrollTargets(opts) : []` with a comment justifying it ('A host with no reference has nothing to track, so it subscribes to nothing: scroll fires on every frame of every scroll and the handler could only ever no-op'). `updated` resolves and attaches the full target set BEFORE its dormant early-return at :103, so the same condition produces opposite subscriptions depending on which hook it arrives through. Probed: zero scroll listeners after a dormant mount, one after a dormant update. And `scheduleUpdate` (schedule-update.ts:24) guards only `enabled === false`, not a missing `to` — so each of those scroll frames runs `calculatePosition`, which deletes three data attributes, removes two custom properties, calls `releaseHide` and rewrites `data-teleport-state` on a host that is not being positioned. Probed: the host's dataset was rewritten by the first scroll frame after a dormant update.

*Symptom:* A closed dropdown whose trigger is `v-if`'d out does six DOM writes per animation frame for as long as the user scrolls, and a consumer MutationObserver on the host sees an attribute record every frame — all for a directive that is doing nothing.

### 10. `scrollContainer` cannot scope anything in the default configuration  ·  `dead`

**src/scroll-target.ts:63**

The listener is capture-phase on `window` (SCROLL_LISTENER_OPTIONS), which already sees scrolls from every nested container in the document — that is the documented reason the default is `'window'`. Since the window floor unions `window` back in whenever `hideWhenReferenceHidden !== false` (the default), naming a `scrollContainer` adds a listener and removes nothing: the handler now fires twice per relevant scroll event (deduped only by the RAF batch), and the option's entire documented purpose ('events elsewhere in the document do not trigger recalcs') is unreachable unless the consumer also turns off a separate, unrelated default. The floor is well argued in the header; what is missing is that it retires the option, and the option's own docs still advertise the behaviour it no longer has.

*Symptom:* Somebody profiling a deep page narrows `scrollContainer` to cut scroll work, measures no change, and goes looking for the bug in their own code.

### 11. The tests for the library's headline claim cannot fail  ·  `flake`

**vTeleportTo.test.ts:5689**

The 'overflow-container escape — README contract' block nests the reference inside `overflow: hidden` / `clip-path` ancestors and asserts the emitted styles match a flat control. jsdom implements no layout and both arms stub `getBoundingClientRect` by hand, so the assertion holds by construction of the fixture — it cannot distinguish a library that escapes clip containers from one that does not. The adjacent 'workaround: useTeleportTo + <Teleport to="body"> slot escapes the transformed containing block' (:5911) is worse: there is no Teleport, no host and no rendering in the test body, only two coordinate assertions that are identical with and without the 'workaround', with a comment explaining what the caller 'will' do. More broadly, only ~50 of 379 tests call `sizeHost`, so ~330 run against a 0×0 host and therefore exercise the projection / `unmeasured` fallbacks rather than the measured paths they are named for.

*Symptom:* 806 green tests are read as evidence that the clip-escape contract and the composable escape hatch hold, while the only things actually asserted are numbers the fixture handed the code. The two measured defects above (horizontal fit, horizontal cross-axis) live in exactly the gap this harness cannot express.

### 12. 585-line function, duplicated cross-axis cascade, two provably-equal booleans, 11 dead guards  ·  `readability`

**src/calculate-position.ts:113**

`computePositionStyles` runs lines 113–697 with 57 top-level locals and does nine jobs in sequence (boundary resolution, two space records, fit, width, coordinates, overflow, hide, arrow, detail). The cross-axis anchor decision is derived TWICE — once in offsetParent coordinates to write `styles.left/right` (:462–490) and once in viewport coordinates to compute `hostLeftViewport` for the arrow (:524–540), plus a third partial copy inside the horizontal shift branch. They agree today only because someone keeps them in step by hand; the one time they did not is TT-16 T2, the arrow that landed 127px outside the popover. `isHorizontal` (from the chosen side, :253) and `isHorizontalPlacement` (from the requested placement, :295) are provably identical for every input, since `resolvePlacement` never crosses axes — two names for one fact, with the mobile full-bleed branch depending on which one you pick. `calculatePosition` then guards 11 style writes with `if (styles.X !== undefined)` for keys the producer always sets, which CLAUDE.md forbids as defensive branches for states the types exclude. The file is 393 comment lines to 407 code lines and most of the prose is changelog ('it used to…', 'that was this library's worst bug'); types.ts is 587 comment lines to 56 code lines; resolve-placement.ts is 186 to 44. For a repo whose product IS the source, a stranger pasting this file must read several hundred lines of archaeology about defects that no longer exist in order to find out what the code now does — and the only two places where the prose is load-bearing (the x-axis 'no loop to close' at resolve-placement.ts:141 and 'safe against the feedback loop' at :339) are the two places it is false.

*Symptom:* The next edit to the cross-axis branch updates one of the two cascades; the arrow silently parts company with the host again, and the behaviour audit passes because the arrow tests run an unsized host and assert the projection.

### 13. measureHostExtent's rect parameter defaults to taking the read the invariant forbids  ·  `dead`

**src/resolve-placement.ts:162**

ARCHITECTURE.md's flagship invariant is 'the host's box is read exactly once per tick', because two reads let the fit test and the arrow disagree about the host's width. The function signature's fourth parameter defaults to `hostEl ? hostEl.getBoundingClientRect() : null` — a second read path that exists, by its own documentation, only 'which is what the unit tests do'. Production always passes the rect, so the invariant holds by convention, and the escape hatch is live, exported and the first thing a copy-paste caller will use because it is the shorter call.

*Symptom:* Anyone reusing `measureHostExtent` in their own code (the stated distribution model for this repo) takes a second layout read per tick and reintroduces the disagreement the invariant exists to prevent, while the tests that are supposed to pin the invariant are the reason the hole is there.

## Comments and docs the code contradicts

- src/resolve-placement.ts:141 — "**Horizontal (`'x'`).** Just the rect's width: the width this library writes … does not depend on which side was chosen, so there is no loop to close". Measured false in Chrome: the rendered width is capped by the left/right coordinate this library writes for the chosen side, which makes the x-axis fit test self-confirming.

- src/calculate-position.ts:339 — "Safe against the feedback loop that governs the height axis: nothing here makes the host's width depend on the side chosen or on its own left edge." Measured false: `crossAxisAlign: 'center'` writes `left` from the measured width, and CSS shrink-to-fit caps the width at `viewport − left` (320→240→200→180→170… in Chrome).

- src/resolve-placement.ts:126 — "a host measured mid-open-animation under-reports — a transient that self-corrects when the animation settles". Nothing re-measures the host: the observers attach to the reference, never to the host (already documented as false in tickets/TT-18).

- src/types.ts:181 — `flip`: "the placement degrades to the comparative rule (pick the side with more room, ties going to the preferred side)" when unmeasurable. resolve-placement.ts:220 keeps the preferred side and its header explains why degrading would be wrong.

- src/use-teleport-to.ts:68 — `fit`: "without it placement falls back to the comparative rule". Same stale rule as types.ts:181.

- src/types.ts:222 — "The `useTeleportTo` composable does not know the host element it will be applied to; in the composable, 'absolute' always uses the viewport fallback." The composable has taken a host argument since the TT-16 T6 fix.

- src/types.ts:435 — `scrollContainer` HTMLElement form: "events elsewhere in the document do not trigger recalcs". The window floor (scroll-target.ts:63) unions `window` in by default, so they do.

- src/use-teleport-to.ts:55 — `placement` is documented as "`null` if disabled or if the reference element is missing/detached"; the missing/detached branch never assigns it.

- src/use-teleport-to.ts:185 — "the composable reports the same 'not hidden by us' answer rather than a stale `true`" — while the `styles` ref it also owns still carries `visibility: 'hidden'`.

- ARCHITECTURE.md:16 — the auto-update row lists its dependencies as `types`, `vue (toValue)`; auto-update.ts:15 imports `isVirtualReference` from `./calculate-position`, and the ASCII graph at :30 omits the module entirely while claiming "No upward edges".

- ARCHITECTURE.md:79 — "a test that forgets to [stub a box] is silently exercising the comparative fallback instead"; an unmeasurable host now keeps the requested side, as the same file states correctly in its resolve-placement row.

- README.md:211 — the data attributes are "cleared when the directive is disabled (`enabled: false`), or when its `to` reference becomes missing or detached". True for placement and fit, false for `data-teleport-collapsed` on the dormant `updated` path.

- playground.smoke.test.ts:14 — "This suite mounts the playground's template structure via createApp(...)". It mounts a hand-written `h()` imitation of four cards; `playground.html` is never read, and the real playground has twelve cards.

## ARCHITECTURE.md claims nothing enforces

- "the host's box is read exactly once per tick" (calculate-position row) — held only by the production caller passing its rect; `measureHostExtent`'s fourth parameter defaults to taking its own `getBoundingClientRect()`, and the tests use that default, so the invariant's only guard is the habit of the one caller.

- "every style key the directive can write is written on every tick, `''` where it does not apply" — true of the style record, but the `data-teleport-*` attributes that mirror it have no equivalent rule, and `data-teleport-collapsed` is left stamped on the dormant `updated` path (directive.ts:103) while its three siblings are cleared.

- "a strictly downward dependency graph … No upward edges" (ARCHITECTURE.md:4, :45) — nothing checks it, the table omits `auto-update → calculate-position`, and the diagram omits auto-update and scroll-target altogether.

- "the directive only clears a hide it itself applied" — enforced via the `data-teleport-hidden` marker, but the value restored is the inline `visibility` captured at hide time, so a consumer who writes `visibility` while the directive's hide is active has that write silently reverted on release.

- "`fit: 'fits'` must keep meaning the host fits on this side" (tickets/TT-19 'must not break') — unachievable on the horizontal axis as written: the extent is measured through the constraint the chosen side imposed, so `fits` is the only answer steady state can produce.

- "`flip` is fit-based, not comparative — the preferred side is held while the host fits on it and moves only when it cannot" (resolve-placement row) — holds as arithmetic, but the input it arithmetises is the previous tick's own output, so the guarantee is only as good as `measureHostExtent`, which is circular on both axes.

## Tests passing for the wrong reason

- vTeleportTo.test.ts:5689 — the whole `overflow-container escape — README contract` block (8 tests). jsdom has no layout and both arms stub the reference rect by hand, so 'ancestor overflow:hidden does NOT change computed top/left' and the nested clip-path variant hold by construction of the fixture. They verify the library's headline claim without being able to observe clipping at all.

- vTeleportTo.test.ts:5911 — 'workaround: useTeleportTo + <Teleport to="body"> slot escapes the transformed containing block'. No Teleport, no host, no rendering; it asserts two coordinates that are identical with and without the workaround, with the actual claim stated only in a comment ('The composable's caller will bind these styles inside a <Teleport to="body"> slot').

- vTeleportTo.test.ts:3145 / 3156 / 3174 / 3182 — crossAxisAlign 'center'/'end' on placement left/right. The host is never sized, so its measured height is 0, and each expected value is derived from `maxHeight/2` — the same expression the code uses. A 40px host is off by 100px and these tests cannot see it.

- vTeleportTo.test.ts:3233 — "arrow tracks crossAxisAlign:'center' on horizontal placement" asserts `--teleport-arrow-y: 120px` on a host whose rect height is 0, i.e. it pins the arrow 120px outside the host as the contract.

- vTeleportTo.test.ts:252 — 'cleans up listeners on unmount' spies on removeEventListener and asserts the call list contains 'resize' and 'scroll'. It never asserts the listener is gone, so a capture-flag or handler-identity mismatch (the classic listener leak) passes.

- Structural: only ~50 of 379 tests call `sizeHost`, so ~330 run with a 0×0 host. Every cross-axis, arrow and overflow assertion in those tests exercises the projected-width / `unmeasured` fallback rather than the measured path it is named for — the harness's `sizeHost` also returns a constant width regardless of where the host was positioned, which is the single assumption that hides the horizontal fit defect.
