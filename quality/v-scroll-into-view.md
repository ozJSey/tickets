# Quality audit — `v-scroll-into-view`

**Verdict: structurally-unsound**  ·  22 findings


## The single worst thing

The container path — the package's headline feature and the only code here that does arithmetic — converts viewport coordinates into scroll coordinates with a formula that is only correct for a container with no border and no transform. `relTop = targetRect.top - containerRect.top + container.scrollTop` measures the origin from the BORDER box while `clientHeight`/`clientLeft` semantics (and `scrollTop` itself) are measured from the PADDING box; `clientTop`/`clientLeft` appear nowhere in the package. Every container scroll is therefore off by the container's border width — always, silently — and off by ~15px on the inline axis in RTL (vertical scrollbar on the left), and off proportionally under any `scale()`/`zoom` between target and container (viewport-space deltas added to layout-space scroll values). What makes it structural rather than a bug is that nothing built to verify it can see it: `makeContainer()` in the test suite cannot express a border at all (it takes rect, clientHeight and scrollTop as independent numbers, so the arithmetic is pinned against a geometry no real element can have — the author's formula checked against the author's formula); the one real-browser check is `Math.abs(l[1] - n[1]) <= 2` under the name "lands on the same pixel as native"; and every playground pane is `.pg-scroller { border: 1px }`, so the error is exactly 1px everywhere it is demonstrated. Card 12 — the card built to prove native parity, whose own prose says "the two panes must land on the same pixel" — prints `top 0` for the directive and `top 1` for native right now, and calls that parity.


## Findings

### 1. Container math mixes border-box origin with padding-box scroll space (no clientTop/clientLeft, no transform handling)  ·  `rot`

**v-scroll-into-view/src/execute-scroll.ts:93**

`relTop = targetRect.top - containerRect.top + container.scrollTop` treats the container's border-box top as the origin of the scrollable area. It is not: scrollTop 0 puts the PADDING edge at the scrollport top, so the correct conversion subtracts `container.clientTop` (and `clientLeft` for the inline axis) — the canonical idiom for exactly this job. `grep -n client src/*.ts` returns only clientHeight/clientWidth: the sizes come from the padding box, the origin from the border box. Error = border width, always, on every alignment and on the `nearest` in/out-of-view test. On the inline axis in RTL the left-placed vertical scrollbar adds ~15px to clientLeft. And because the rect deltas are viewport-space while scrollTop/clientHeight are layout-space, any `transform: scale()` or `zoom` between target and container scales the error by (1-s) × distance — 300px wrong for a 600px jump inside a 0.5-scaled modal. Native scrollIntoView gets all three right, which is why the README's parity claim is false.

*Symptom:* `block:'start'` in a bordered pane hides the top border-width of the target behind the border — a hairline clip at 1px, a visible slice at 4px. `nearest` judges a target sitting within a border-width of the top edge as out of view and scrolls when it should not. In an RTL list, horizontal alignment lands ~15px off. Scroll into view fired while a modal is mid-scale-animation lands tens to hundreds of pixels off, non-reproducibly.

### 2. The only real-browser parity check is tolerance-blind to the defect it was written to catch  ·  `rot`

**playground/scripts/interactions/v-scroll-into-view.mjs:231**

The check named "B2 nearest: a target taller than the pane lands on the same pixel as native" asserts `Math.abs(l[1] - n[1]) <= 2`. The file's own header argues that parity asserted against the browser beside it "is a measurement" rather than "a number someone chose" — then chose 2. The coordinate-system error above is exactly 1px in every demo (`.pg-scroller` carries `border: 1px`), so it lives permanently inside the tolerance. Worse, the demo's readout measures `target.top - pane.top`, which is taken from the same wrong origin, so the buggy library value reads as a rounder number (`top 0`) than the correct native one (`top 1`) — a reader eyeballing card 12 would conclude the directive is the more precise of the two. Every other assertion in the file uses the same ±2, so the whole harness is structurally unable to see any error smaller than a 3px border.

*Symptom:* The repo believes native parity is measured and green. It is asserted with a tolerance wider than the defect, on a demo whose own readouts visibly disagree.

### 3. `block:'end'` subtracts a leading-edge gap from a trailing-edge alignment, and a test pins it as correct  ·  `rot`

**v-scroll-into-view/src/execute-scroll.ts:51**

`if (align === 'end') return far - client - offsetStart`. `offsetStart` is documented in this very file as "the leading `offsetStart` pixels of the scrolling box are treated as obscured by a sticky header" — a description of the TOP edge. Applied to an end-alignment it opens a gap at the BOTTOM, where nothing is obscured. The native path disagrees: `scroll-margin-top` has no effect on `block:'end'` (it expands only the top of the scroll-margin box), so the same options produce a 64px bottom gap with `container` and none without it. README line 246 claims the alignments agree "with two deliberate exceptions"; this is an undocumented third. The unit test at vScrollIntoView.test.ts:748 ("offset still subtracts from final scrollTop") cements the divergence as intended without ever comparing against native.

*Symptom:* A chat pane with a sticky header and a global `offset: { top: 64 }` pinned with `block:'end'`: the newest message never reaches the bottom of the pane, it rests 64px above it, and nothing in the docs explains the gap.

### 4. `block:'center'` + offset shifts by the full offset — an arbitrary third answer with zero tests  ·  `rot`

**v-scroll-into-view/src/execute-scroll.ts:52**

`return rel + size / 2 - client / 2 - offsetStart`. Centring inside the unobscured region [offsetStart, client] requires shifting by `offsetStart / 2`; ignoring the offset for a centred alignment is also defensible. Shifting by the full offset is neither: the target ends up offsetStart/2 below the true centre of the visible region. `grep -n center vScrollIntoView.test.ts` shows `center` is never once combined with `offset` in either harness — it is the only alignment×offset pair with no coverage anywhere, and the one whose formula is a guess.

*Symptom:* `block:'center', offset:{top:64}` lands the target 32px low. Nobody measures a centre, so it reads as "the library is slightly off" rather than a bug with an address.

### 5. Edge detection records the condition as consumed even when the scroll was refused  ·  `rot`

**v-scroll-into-view/src/directive.ts:78**

`updated()` writes `state.previousCondition = opts.condition` and then calls `doScroll`. `executeScroll` returns `void` and bails silently for a target with no layout box (execute-scroll.ts:84) or a container that resolves to null/detached (execute-scroll.ts:88) — there is no path by which the scheduler that owns the edge state can learn the scroll did not happen, so it cannot re-arm. The false→true edge is spent on a no-op and there is no second chance: while the condition stays true, no further update will ever scroll. The bookkeeping is driven by the binding value rather than by whether the work was performed.

*Symptom:* Mount a panel with `condition: true` while an ancestor is still `display:none`, or with `container: '#pane'` where `#pane` is behind a `v-if` that resolves a tick later, and the element never scrolls — for the whole lifetime of the component. Playground card 11 dodges this by instructing the reader to press the button a second time.

### 6. `container` is never checked for ancestry or scrollability, and a plain selector matches document-wide  ·  `rot`

**v-scroll-into-view/src/resolve.ts:52**

`resolveContainer` returns whatever `document.querySelector` / `el.closest` / the getter hands back. Nothing asserts `container.contains(el)` (the type and both docs call it a "Scrollable ancestor"), and nothing asserts it can scroll. `document.querySelector('.pane')` returns the FIRST `.pane` in the document, so in a multi-pane layout every row in every pane resolves to pane #1 and the arithmetic computes a position relative to an element the target does not live in. `:scope` exists to fix this but is opt-in and the README presents the plain selector first. A non-scrollable container is a total silent no-op (scrollTo clamps to 0) and the directive deliberately refuses to fall back to native, so the most common misconfiguration produces silence with no dev warning anywhere in the package.

*Symptom:* Two chat panes on one screen: selecting a message in the second pane yanks the FIRST pane to a nonsense offset and leaves the real target where it was. Or, with the overflow on an inner div, nothing happens at all, forever, with no console output to search for.

### 7. A scroller between target and container is never scrolled, so the target can stay clipped  ·  `rot`

**v-scroll-into-view/src/execute-scroll.ts:120**

Only the named container is scrolled. Native `scrollIntoView` walks and scrolls every scrollable ancestor; pinning a container opts out of all of them, including ones BELOW the container that are clipping the target. The geometry stays internally consistent, so the directive reports success by scrolling the outer pane to a position where the target is still invisible inside an inner scroller. The README addresses only the reverse, easy case ("pass the inner container and the outer one is left untouched") and the unit test at vScrollIntoView.test.ts:1344 picks that same easy direction.

*Symptom:* "It scrolls to the row most of the time" — failing exactly when the inner pane happens to be scrolled away, which users cannot describe and bug reports cannot reproduce.

### 8. `offset: { top: 0 }` silently deletes the consumer's CSS scroll-margin on the native path  ·  `rot`

**v-scroll-into-view/src/execute-scroll.ts:129**

The native path branches on `off?.top !== undefined`, not on whether a gap was actually requested, then writes `scrollMarginTop = '0px'` inline across the call. An inline `0px` overrides a stylesheet `scroll-margin-top: 64px` for the duration of the scroll, so passing a zero offset actively disables the CSS gap rather than leaving it alone. The test that covers this is named "offset zero treated as legitimate value (subtracts 0, no-op math)" (vScrollIntoView.test.ts:660) and only exercises the container path, where it genuinely is a no-op.

*Symptom:* `offset: { top: headerHeight }` with the header collapsed to 0 loses the global sticky-header gap that card 13 exists to advertise — the heading slides under the header only when the header is hidden, which is the opposite of intuition.

### 9. The README's table-of-contents recipe works only because a falling edge does not cancel the pending frame  ·  `flake`

**v-scroll-into-view/src/directive.ts:80**

`if (!opts.condition) return` returns without touching `state.pendingRaf`, so a condition that goes true and back to false inside one frame still scrolls. README:140 ships exactly that shape as the canonical anchor-navigation recipe — `activeId = id` then `requestAnimationFrame(() => activeId = null)` — and it only scrolls because the directive's queued frame survives the condition being withdrawn microseconds earlier. Nothing documents the dependency and no test pins it; the obvious-looking hardening ("cancel the pending scroll when the condition goes false") silently breaks the package's own documented recipe and five playground demos that copy the same false→rAF→true ritual.

*Symptom:* A future one-line "fix" in `updated` makes every documented trigger pattern stop working, with green tests, because the behaviour they depend on is an omission rather than a decision.

### 10. `nearest` decides visibility against a mid-animation scroll position and then issues no scroll at all  ·  `flake`

**v-scroll-into-view/src/execute-scroll.ts:64**

`scrollFor` compares the target against `container.scrollTop` read live. With `behavior:'smooth'` (the default) a previous scroll is still animating, so that value is a moving coordinate that has not reached its destination. When the target reads as currently visible, the function returns `null`, `executeScroll` returns without calling `scrollTo` at all, and the in-flight animation continues to the OLD destination — the new request is not merely late, it is discarded. Both halves of the trap are defaults (`nearest` + `smooth`), and jsdom can never expose it because `scrollTo` is a mock that does not move `scrollTop`.

*Symptom:* Hold the arrow key through a long list: the active row drifts toward the edge and intermittently ends up off-screen, differently every time, depending on where the animation happened to be in the frame the decision was made.

### 11. `matchMedia` is evaluated on every update of every bound element, not "at scroll time" as documented three times  ·  `lie`

**v-scroll-into-view/src/resolve.ts:19**

types.ts:23 says the reduced-motion default is "read at scroll time", resolve.ts:15 says "Read fresh rather than cached, so a preference changed mid-session takes effect on the next scroll", README:298 repeats it. The read actually happens inside `resolveBinding`, which `updated()` calls on EVERY re-render of every element carrying the directive, whether or not a scroll follows — the scroll itself happens a frame later in the rAF, where the value is never consulted. The docs describe the better design; the code pays for it on the wrong trigger.

*Symptom:* A 40-row list (playground card 02) evaluates 40 media queries per keypress; a virtualised 1000-row table evaluates 1000 per re-render, for a value that matters only on the one row that will actually scroll.

### 12. resolve.ts's hardening comment is contradicted by the code one line below it  ·  `lie`

**v-scroll-into-view/src/resolve.ts:27**

The comment claims "Anything that isn't a boolean, plain object, or `undefined` falls back to 'disabled' defaults". `isObj` is `value !== null && typeof value === 'object'`, which is true for arrays, Dates, Maps and any class instance — all of them take the options branch, where `obj.condition ?? true` resolves to TRUE. `v-scroll-into-view="items"` (a plausible slip for `items.length > 0`) scrolls on mount. The README is narrower and stays accurate by listing only null/numeric/string, which makes the source comment the one that lies.

*Symptom:* A binding typo that should be inert scrolls the page on mount, and the comment directly above the code tells the next reader that cannot happen.

### 13. A test section advertises an "RTL probe" that does not exist, and the package has no RTL handling at all  ·  `lie`

**v-scroll-into-view/vScrollIntoView.test.ts:1203**

The describe banner reads "Edge cases — container + offset + `nearest`, RTL probe, behavior:'auto'". There is no RTL test in that block or anywhere else: `grep -rin "rtl|direction|writing-mode"` across the whole package returns this one comment. Meanwhile `scrollFor` hard-codes LTR semantics — `start` is always the left/top edge and scrollLeft is assumed non-negative, where RTL scrollLeft is 0..-max and native `inline:'start'` means the RIGHT edge. A reader auditing coverage sees the word RTL and moves on.

*Symptom:* In an RTL app the container path scrolls `inline:'start'` to the wrong edge entirely, while the container-less path (the browser) gets it right — so the same option behaves oppositely depending on whether `container` is set.

### 14. playground.smoke.test.ts: 21 tests that are 7 assertions × a viewport the library never reads, on an app with no directive in it  ·  `rot`

**v-scroll-into-view/playground.smoke.test.ts:159**

`setViewport()` writes `window.innerWidth/innerHeight`; nothing in the package reads either, so the three "breakpoints" run byte-identical assertions and cannot differ — 14 of the 21 tests are pure weight (×2 for the Vue-version matrix = 28 of 42 executions). The docblock claims the suite "mounts a Vue app via createApp(...).use(ScrollIntoViewPlugin).mount()" and "exercises the directive's public surface", but `mountPlayground` renders items with NO directive on any element (lines 85-104) and every hook is then hand-invoked through `as any` (lines 128-157), so Vue never drives the directive and the `updated` hook's real trigger — a parent re-render — is never exercised anywhere in the package. The file is named for `playground.html`, which it never loads, and calls it "the canonical 'does the library work in a real Vue app' exhibit" while playground.html's own comment says to use `../playground` to judge behaviour.

*Symptom:* "272/272 tests across 5 workspaces" (the manifest's status line) is ~120 distinct test bodies, none of which prove Vue ever calls this directive correctly.

### 15. The rAF teardown-race guard has zero coverage; the test that claims to cover it proves the opposite  ·  `rot`

**v-scroll-into-view/vScrollIntoView.test.ts:1561**

The test comment says "the rAF cb still gets invoked manually (paranoia: browsers may race the cancel)" — but nothing invokes it: `triggerUnmount` calls the mocked `cancelAnimationFrame`, which filters the callback out of `rafCallbacks`, so the subsequent `flushRaf()` iterates an empty queue. `expect(() => flushRaf()).not.toThrow()` passes vacuously and `not.toHaveBeenCalled()` passes because the cancel worked, never because the `!stateMap.has(el)` guard fired. The guard itself (directive.ts:23-27) is defensive code for a state `cancelAnimationFrame` makes impossible, justified by a comment asserting browser behaviour that does not exist — against CONVENTIONS' "No defensive programming for impossible states". `doScroll`'s `if (!state) return` is likewise unreachable from both call sites.

*Symptom:* An untested branch and two dead guards in the file strangers copy, each carrying a comment that teaches the reader something false about rAF.

### 16. A second, divergent copy of the directive ships in the package root — the exact thing ARCHITECTURE.md's invariant forbids  ·  `rot`

**v-scroll-into-view/playground.html:169**

ARCHITECTURE.md:19 states "The invariant that matters: `execute-scroll.ts` is the only place a scroll happens... splitting the file must never reintroduce a second copy." playground.html contains a hand-written reimplementation of resolveBinding, doScroll and all three lifecycle hooks, which its own comment admits is "the pre-1.2.0 `nearest` behaviour" with no container, no offset and no reduced-motion default. Nothing enforces the invariant: not a test, not a lint rule, not the build. It is a wish, and it is already violated in the same directory — by the most self-contained, most copy-pasteable file in the package (it runs from a CDN with no build step).

*Symptom:* A stranger skimming the repo copies the one file that runs standalone and ships the 2026-05 `nearest` bug that CHANGELOG 1.2.0 says was fixed.

### 17. `condition` and `always` are inert in the composable but are part of the options type it accepts  ·  `dead`

**v-scroll-into-view/src/use-scroll-into-view.ts:91**

`resolveBinding({ condition: true, ...lastOpts })` — `lastOpts` spreads last, so a consumer's `condition: false` wins, and `executeScroll` never reads `opts.condition` at all. So the leading `condition: true` is dead AND `useScrollIntoView({ target, options: { condition: false } }).scroll()` scrolls anyway. `always` is edge-detection state that only `directive.ts` consults, equally inert here. Both are advertised: the params type is `options?: VScrollIntoViewOptions` and README:400 describes that type as "Options accepted by directive binding + composable".

*Symptom:* A consumer gating the composable with `update({ condition: isOpen })` gets an unconditional scroll, with full TypeScript approval.

### 18. `ContainerRef` is exported from types.ts but dropped by the public barrel  ·  `dead`

**v-scroll-into-view/src/index.ts:9**

types.ts:8 exports `ContainerRef`, the union that describes the whole `container` feature. Neither src/index.ts nor the build entry re-exports it, so a consumer who wants to type a helper returning a container has to hand-copy `HTMLElement | string | (() => HTMLElement | null)` — which is exactly what README:385 does in the options table. The Exports table lists eight names and this is not one of them.

*Symptom:* Playground card 03's `function container()` has no importable return type; every consumer retypes the union or reaches into the package's internals.

### 19. The decision that gates the entire library is a four-level nested ternary serving inputs the types exclude  ·  `readability`

**v-scroll-into-view/src/resolve.ts:34**

`isBool ? value : isObj ? (obj.condition ?? true) : value === undefined ? true : false` is the single most consequential expression in the package (scroll or not) and the least readable line in it. Its whole shape exists to service `null`/number/string bindings that the declared type `boolean | VScrollIntoViewOptions | undefined` already excludes and that `vue-tsc` (which the playground runs) would reject — defensive programming for impossible states, which CONVENTIONS:40 bans outright, rewarded with a playground card, three unit tests and a README paragraph. It is also where the array inconsistency hides: the hardening is not even uniform across the cases it was written for.

*Symptom:* A stranger copying resolve.ts reads a 7-line conditional to answer "when does this scroll?", and the answer it gives for an array contradicts the comment four lines above it.

### 20. The native path is written in two-letter locals with non-null assertions it creates for itself  ·  `readability`

**v-scroll-into-view/src/execute-scroll.ts:128**

Fourteen lines holding `off`, `st`, `sl`, `s`, `pt`, `pl` — then `off!.top` and `off!.left`, because storing the discriminant in a separate boolean (`st`) threw away the narrowing TypeScript had already done on `off?.top !== undefined`. The `!` assertions are self-inflicted: inline the check and they disappear. In a repo whose first convention is "no escape hatches" and whose distribution model is people reading these files, the save-then-restore block that a reader most needs to follow is the one written in initials.

*Symptom:* The block where a copy-paste consumer is most likely to introduce a leak (writing inline style on someone else's element) is the hardest one to read, and it models `!` as acceptable style.

### 21. The composable's showcase, in both README and playground, is a permanently disabled button  ·  `readability`

**playground/src/demos/v-scroll-into-view/07-composable.vue:28**

`:disabled="scroller.state.value === 'idle'"` on the `cancel()` button, with the prose underneath admitting the button "is almost always disabled — that is the point of the reactive state". README:352 ships the identical snippet and README:368 makes the same admission. The demonstration of `cancel()` and of the reactive `state` is therefore a control that a human can never successfully click, presented twice as the intended usage. `state` is 'pending' for exactly one frame by construction, so nothing a user drives can key off it.

*Symptom:* A stranger pastes the example, sees a dead button, and concludes the composable is broken — or ships the dead button.

### 22. Demo 05 teaches `always: true` as the chat-follow pattern without the one check that makes it usable  ·  `readability`

**playground/src/demos/v-scroll-into-view/05-always.vue:27**

`{ condition: true, always, container: '#always-pane', block: 'end' }` re-scrolls on EVERY `updated` call — i.e. every re-render of the component for any reason, not just a new message — and there is no "is the user already near the bottom?" guard, which is the entire difficulty of chat autoscroll. The card frames this as the chat log recipe.

*Symptom:* Paste it into a real chat and the user cannot scroll up to read history: any unrelated re-render snaps the pane back to the bottom.

## Comments and docs the code contradicts

- v-scroll-into-view/src/resolve.ts:27 — "Anything that isn't a boolean, plain object, or `undefined` falls back to 'disabled' defaults": arrays, Dates and class instances all satisfy `typeof === 'object'`, take the options branch and resolve to condition TRUE.

- v-scroll-into-view/src/types.ts:23 and src/resolve.ts:15 and README.md:298 — the reduced-motion default is "read at scroll time" / "on the next scroll": it is read in `resolveBinding`, on every mounted/updated of every bound element, and never consulted in the frame where the scroll happens.

- v-scroll-into-view/src/directive.ts:23 — "if `unmounted()` raced ahead of cancelAnimationFrame (browser queued the cb before seeing the cancel)": cancelAnimationFrame is specified to remove the callback; the state it guards against is not reachable, and no test reaches it.

- v-scroll-into-view/README.md:246 — "The alignments agree ... with two deliberate exceptions, both documented below": there are at least three more (the border-width/transform coordinate error, `block:'end'` + `offset.top`, and `scroll-padding-*` on the scrollport, which the container path never reads and no doc mentions).

- v-scroll-into-view/README.md:314 — "'pending' while the rAF is queued, then 'idle' after the scroll fires": directive.ts:28 sets 'idle' immediately BEFORE calling executeScroll, not after.

- v-scroll-into-view/playground.smoke.test.ts:5 — "mounts a Vue app via createApp(...).use(ScrollIntoViewPlugin).mount() and exercises the directive's public surface": the mounted component has no directive on any element; all hooks are hand-invoked through `as any`. The same docblock calls playground.html "the canonical 'does the library work in a real Vue app' exhibit" while playground.html:172 says to use ../playground instead — and the test never loads playground.html at all.

- v-scroll-into-view/vScrollIntoView.test.ts:1563 — "the rAF cb still gets invoked manually": the mocked cancelAnimationFrame removed it from the queue, so flushRaf() invokes nothing.

- v-scroll-into-view/vScrollIntoView.test.ts:1203 — describe banner advertises an "RTL probe" that does not exist anywhere in the package.

- playground/src/demos/v-scroll-into-view/12-nearest-oversized.vue:108 — "the two panes must land on the same pixel — that is what 'native parity' means": with the pane's 1px border they land 1px apart, and the card's own readouts print `top 0` vs `top 1`.

- playground/scripts/interactions/v-scroll-into-view.mjs:231 — check named "lands on the same pixel as native" asserts `<= 2`; the file's header simultaneously argues that a chosen number is not a measurement.

## ARCHITECTURE.md claims nothing enforces

- ARCHITECTURE.md:19 'execute-scroll.ts is the only place a scroll happens ... splitting the file must never reintroduce a second copy' — playground.html:169-220 is a second copy of resolveBinding + doScroll + all three hooks, self-described as the pre-1.2.0 nearest behaviour. Nothing (test, lint, build) prevents or detects it.

- ARCHITECTURE.md:29 ''offset' is folded into scrollFor, never applied afterwards' — true today, but nothing stops the next edit from subtracting post-hoc again; no test asserts the property, only specific numbers.

- ARCHITECTURE.md:34 'every file under src/ plus the entry is self-contained TypeScript ... take the folder as-is' — package.json files is ["dist","CHANGELOG.md"], so the npm tarball has no src/, and the repository/homepage links point at github.com/ozJSey/vue-scroll-into-view, which per the root CLAUDE.md does not exist as a remote. The documented distribution channel for the thing that IS the product is unreachable.

- types.ts:53 / README:209 'container always wins when set' is enforced, but the companion promise that container is a 'Scrollable ancestor' is enforced by nothing — no contains() check, no scrollability check, and document.querySelector resolves document-wide.

- README:13 'the native alignment rules, nearest included, applied to the container you pick' — no test or check compares the container path to native for start/center/end, and the one nearest comparison runs at ±2px.

## Tests passing for the wrong reason

- vScrollIntoView.test.ts:1561 'rAF callback resilient to mid-flight unmount' — the callback is never invoked (cancel removed it from the mock queue), so the `!stateMap.has(el)` guard it names has zero coverage; both assertions pass vacuously.

- vScrollIntoView.ssr.test.ts:151 'back-compat type alias ScrollIntoViewOptions equals VScrollIntoViewOptions' — self-confessed in its own comment: "Runtime asserts a tautology so the test passes."

- playground.smoke.test.ts:159 — the three viewport describes are 7 identical assertions repeated; nothing in the library reads innerWidth/innerHeight, so 14 of the 21 tests cannot ever differ from their siblings. ×2 for the Vue-version matrix.

- vScrollIntoView.test.ts:280 makeContainer() — the fixture takes rect, clientHeight and scrollTop as independent numbers and cannot express a border or a transform, so every container-math test pins the arithmetic against a geometry no real element can have. This is the fixture that makes the clientTop defect undetectable.

- vScrollIntoView.test.ts:660 'offset zero treated as legitimate value (subtracts 0, no-op math)' — true on the container path only; on the native path `{top:0}` writes an inline 0px that suppresses the consumer's CSS scroll-margin-top, which the test never exercises.

- vScrollIntoView.test.ts:748 'container + offset with block: end: offset still subtracts from final scrollTop' — pins a behaviour that diverges from the native path (scroll-margin-top has no effect on an end alignment) as if it were parity.

- vScrollIntoView.test.ts:1344 'nested scroll containers: only the selected container scrolls' — exercises the easy direction (pass the inner container). The failing direction, an inner scroller clipping the target while the outer one is scrolled, is untested.

- Whole directive suite — every hook is called by hand through `as any` (49 casts across the three test files); the app built in mountWithValue() at line 46 is never mounted and nothing in the package is ever driven by Vue's own update cycle.

- vScrollIntoView.test.ts — the hidden-target guard only behaves as in production when a test calls pretendLayoutEngine(); every other test in the file runs with the guard disabled by jsdom's empty getClientRects, including the one that asserts that (line 1789).
