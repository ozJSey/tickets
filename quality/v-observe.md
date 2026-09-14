# Quality audit — `v-observe`

**Verdict: structurally-unsound**  ·  18 findings


## The single worst thing

Every option that decides WHAT gets observed (`intersect.root`, `intersect.rootMargin`, `intersect.thresholds`, `resize.box`) is read exactly once, inside the observer constructor call, and is then unreachable forever — while every option that decides what gets REPORTED is re-read live from `internal.cfg` on each callback. Nothing in the code, the types, the README or the tests distinguishes the two halves, so the package looks uniformly reactive and is half frozen. The snapshot is taken at the one moment the values are guaranteed to be wrong: Vue evaluates the binding object during render, before template refs are assigned, so `root: scroller` is `null` at `mounted` (proved: mounted sees `root=null`, `updated` sees the element, and `setupIntersect`'s early return at src/intersect.ts:86-90 only copies `cfg` and never rebuilds the observer). Consequence: the README's "Nested scroll container" recipe and all seven playground demos that pass `root: scroller` observe the viewport, not the pane, and `resize.box` never reaches `observe()` at all. It degrades quietly — intersection is still clipped by overflow ancestors, so lazy-load "works" and nobody looks again — and a copied file can never be patched.


## Findings

### 1. Observer construction options are a one-time snapshot taken before template refs exist; the later correct value is accepted into cfg and ignored  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/intersect.ts:102**

`observerOptions` (root / rootMargin / threshold) is built once and handed to `new IntersectionObserver`. Both swap paths — src/directive.ts:44 and the early return at src/intersect.ts:86-90 — only assign `state.intersect.cfg` and return, so the observer is never rebuilt. I verified with real Vue + jsdom that a binding written `{ intersect: { root: scroller } }` delivers `root=null` to `mounted` and the element only to `updated`: the render that produces the object runs before the ref is set. So `root` is permanently null for the documented usage, `rootMargin` is permanently the mount-time string (playground 05's slider is inert), and `thresholds` changes alter the `crossed` math at src/intersect.ts:21 while the observer keeps sampling at the old threshold set — the crossing math is then fed ratios from a granularity that no longer matches its thresholds.

*Symptom:* "My lazy-load / reveal fires when the page scrolls instead of when my scroll-pane scrolls", and a rootMargin preload distance that has no effect. No error, and intersection is still clipped by overflow ancestors so the feature looks 90% right.

### 2. `mutate: { on: 'attr:*' }` is an unbounded MutationObserver feedback loop — the directive observes an attribute it writes itself  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/mutate.ts:60**

`flushMutateEvents` calls `writeStateAttribute(el, …)` on the host, inside the mutate dispatch path, and `setAttribute` queues a MutationRecord even when the value is unchanged. With `attributes: true` and no filter (src/mutate-records.ts:38-40) that record comes straight back as `attr:data-observe-state`, flushes again, writes again. Confirmed with a real MutationObserver: one external `setAttribute` produced `attr:data-kick` followed by an endless run of `attr:data-observe-state` — it only stopped because I force-unmounted at 200 calls; the unguarded version hung the test process until it was killed at 2 minutes (microtask starvation = frozen tab). The 150 ms `mutate:active`→`idle` cooldown write re-arms it on its own. The playground ships the trigger as a dropdown option (playground/src/demos/v-observe/11-mutate-attr.vue:24) and the README advertises `attr:*` as a feature. `writeStateAttribute` has no "did the string change?" guard, which is both the loop's enabler and an unnecessary style invalidation on every single intersect/resize tick.

*Symptom:* Tab freezes solid, no stack, no error — the user selects `attr:*` (or subscribes to `attr:data-observe-state`) and the page dies on the next attribute change anywhere on the host.

### 3. `resize.box` never reaches `observe()`, so the option changes what is reported but not what is watched  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/resize.ts:274**

`observer.observe(el)` is called with no second argument — I printed the options the observer actually receives: `undefined`. The observed box is therefore always content-box, which is what decides when a callback fires. `box: 'device-pixel'` exists for exactly one job (re-rastering a canvas when devicePixelRatio changes on zoom or a monitor move) and that job produces NO content-box change, so no callback ever arrives; README recipe 11 is the broken case, verbatim. `box: 'border'` likewise misses border/padding-only changes. The test mock's `observe(el)` signature (vObserve.test.ts:120) takes no init at all, so the suite structurally cannot see this.

*Symptom:* "My canvas goes blurry after the user zooms and never re-renders." The handler is correct, the dimensions are correct when they arrive, and they never arrive.

### 4. `once: true` collapses only if you also passed an `on` callback — README says the opposite  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/intersect.ts:148**

`internal.hasFired = true` lives inside `if (live.on)`. The teardown check at src/intersect.ts:162 is `if (live.once && internal.hasFired)`. With `{ once: true, thresholds: [0.5], crossed: fn }` — no `on` — `hasFired` stays false forever, the observer is never disconnected and `crossed` keeps firing. Verified: disconnected=false, crossed fired `0.5:up 0.5:down 0.5:up` across three ticks. The whole lifetime of the observer depends on the presence of an unrelated reporting callback rather than on the visibility transition. README:562 states "`crossed` events also stop — `once` is the explicit collapse signal" and the shipped d.ts says "Auto-disconnect after the first isIntersecting === true callback".

*Symptom:* A `once` sentinel that keeps calling `loadNextPage()` on every scroll past it, plus an observer that is never released for the life of the page.

### 5. `gateOnIntersect` silently stops gating forever once intersect's `once` collapse runs  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/gate.ts:23**

`teardownIntersect` sets `state.intersect = undefined` (src/intersect.ts:177). `isGated` then hits `if (!i) return false` — the branch documented at src/gate.ts:14-15 as "intersect setup never wired up (e.g. IntersectionObserver unavailable)". So `{ intersect: { once: true, on }, resize: { gateOnIntersect: true } }` gates until the first visible tick and never again, including after the element scrolls far off-screen. `validateOptions` happily accepts the combination. The gate's behaviour also depends on whether you passed an `on` (see previous finding): without one, intersect is never torn down and the gate keeps working — same two options, opposite lifetimes.

*Symptom:* "gateOnIntersect stopped suppressing work" — the expensive handler runs for off-screen elements with no indication the gate is gone.

### 6. With no IntersectionObserver the state attribute reports `intersect:hidden` forever, so CSS-only consumers render permanently invisible content  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/state-attribute.ts:20**

`initialSegments` writes `hidden` whenever intersect is configured; `setupIntersect` returns at src/intersect.ts:83 when the global is missing, and nothing else ever writes that segment. Verified: `intersect:hidden;resize:-;mutate:-` with IO deleted. README recipe 5 (and playground 04) teach `.card { opacity: 0 }` + `[data-observe-state*='intersect:visible'] { opacity: 1 }` — on any engine without IO that content is invisible, permanently. The gate deliberately fails OPEN in the same situation (src/gate.ts:14-16, "graceful degradation prefers always fire"); the CSS hook fails CLOSED, and the suite has a test at vObserve.test.ts:1106 that names `hidden` "the false-positive" for precisely this reason, then another at vObserve.test.ts:3262 that walks past the false positive without asserting the attribute.

*Symptom:* Blank page / invisible cards on an older or locked-down browser, with the JS handlers also silently never firing.

### 7. A gated resize mode loses the ResizeObserver's initial observation and never asks for another measurement  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/resize.ts:251**

`if (isGated(state, live)) continue` drops the callback entirely. Per the HTML update-the-rendering steps, resize observations are gathered BEFORE intersection observations are delivered, so with `gateOnIntersect: true` the mandatory first RO callback is always dropped, even for an element that is visible at mount. `onIntersectVisibilityRestored` then clears the baseline (src/gate.ts:33-43) but does nothing to obtain a fresh size — no `unobserve`/`observe`, no manual read — and RO will not spontaneously re-fire for an element whose box did not change. So the consumer gets no dimensions at all until a real resize happens, possibly never. Playground 16's "resize ticks" counter can only ever read 0 for this reason; nobody noticed because the demo does not throw and v-observe has no entry in the interactions coverage ratchet.

*Symptom:* "My gated resize handler never fires until I drag the window" — layout that depends on the first tick never initialises.

### 8. The resize CSS segment and tick `bracket` are always computed from width, contradicting the `crossed` events they sit next to  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/resize.ts:210**

`bracketLabel(dims.width, …)` and `normalized.labels[bracketIndex(dims.width, …)]` ignore `cfg.axis`. Verified with `axis: 'height'`, breakpoints `{narrow:0, wide:500}`, size 300x700: the handler receives `bracket: 'wide'` while the element's attribute simultaneously reads `resize:narrow`. A stylesheet keyed on `[data-observe-state*='resize:wide']` and a JS log of `e.bracket` disagree at the same instant. Separately, `emitCrossingsForAxis` stamps every event in a multi-bracket jump with the FINAL label (src/resize.ts:133): a 200px→800px jump over [320, 640] emits `[320, '>=640']` and `[640, '>=640']`, so the threshold-320 event claims a bracket the element was never in at that crossing — README:477 calls that field "bracket label entered".

*Symptom:* CSS and JS disagreeing about the active breakpoint on the same element; a state machine keyed on `e.bracket` skipping its intermediate state.

### 9. `observer: null as unknown as IntersectionObserver` publishes a half-built state bag, turning one consumer mistake into a second misleading crash  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/intersect.ts:94**

`state.intersect = internal` happens before `new IntersectionObserver(...)`, and the `observer` field is a cast lie (src/resize.ts:233 is identical). A real `IntersectionObserver` constructor throws RangeError for a threshold outside [0,1] and SyntaxError for a malformed `rootMargin` — both are consumer-supplied strings/numbers, and `rootMargin` is template-interpolated in playground 05. Verified: `thresholds: [0, 1.5]` yields the constructor error AND then `Cannot read properties of null (reading 'disconnect')` on unmount, because `teardownIntersect` dereferences the null. CLAUDE.md/CONVENTIONS ban exactly this cast class; here it converts a clear "your threshold is invalid" into a second error pointing at teardown.

*Symptom:* Two errors for one mistake, the louder of which (`null.disconnect`) points at the library's teardown and sends the reader hunting for a lifecycle bug that does not exist.

### 10. `on: 'text'` reports the changed text node's content under the host's name, can pair `from` and `to` from two different nodes, and misses Vue's most common text patch  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/mutate-records.ts:155**

`to` is `textTarget.textContent` — the last characterData record's node — while `target` is always the host and `from` is the FIRST record's `oldValue`. In a batch touching two text nodes the event is a diff between two unrelated strings, and the payload has no field naming which node changed, so the consumer cannot tell. With `subtree: true` on a multi-node host, `to` is a fragment of the visible text, not the text. Worse, the subscription only covers characterData: Vue compiles `<div>{{ msg }}</div>` to `el.textContent = …`, which is a childList mutation, so the single most common Vue text update produces no `text` event at all. README recipe 15 and playground 13 are a contenteditable validator (`valid = e.to.trim().length >= 5`) — the first Enter keypress splits the content into multiple nodes and the validator starts judging one line.

*Symptom:* "Validation says too short when the box is full", and text bindings that never notify at all.

### 11. `ResizeBracketEvent.from` and `ResizeOrientationEvent.from` are typed nullable but the code cannot emit null, and a demo ships the unreachable branch  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/types.ts:82**

Crossed events are emitted only inside `if (prev && normalized)` (src/resize.ts:176) with `from: prev`; orientation events only when `lastOrient !== null` (src/resize.ts:191). Both `| null` arms are unreachable. README:482 annotates the orientation one "null until the second orientation", which inverts the truth — there IS no event until the second orientation. playground/src/demos/v-observe/09-resize-orientation.vue:12 renders `e.from ?? '(initial)'`, a string that can never appear, and the box shows '—' with no data-orientation until the user resizes it.

*Symptom:* Consumers writing an initial-state branch for an event that never arrives, and the orientation CSS hook being unset on first paint with nothing in the docs explaining it.

### 12. `updated` re-implements the cfg-swap that each setup function already performs, and takes a third, different path for mutate  ·  `readability`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/directive.ts:42**

`state.intersect.cfg = next.intersect` duplicates src/intersect.ts:87-89; `cfg` + `normalizeBreakpoints` duplicates src/resize.ts:226-227; mutate skips the branch entirely and calls `setupMutate` unconditionally on every re-render (re-normalizing and re-keying on each parent render). So the same decision is encoded twice with one exception, and the file's own docstring promises a uniform "setup / cfg-swap / teardown" diff. Any future edit to a setup function's swap path silently will not apply on update. The redundant `if (state)` re-checks at lines 50/62/69 sit inside branches already guarded by `state?.x`.

*Symptom:* A later fix added to setupResize's swap path (e.g. also resetting a baseline) appears to work on mount and silently does nothing on a binding update.

### 13. `computeOrientation` calls a 0x0 box 'square', so hiding an element emits a phantom orientation flip  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/resize.ts:68**

`if (w <= 0 || h <= 0) return 'square'` — a degenerate box is not square, it is unmeasured. ResizeObserver fires with 0x0 when an element goes `display: none`, so a v-if/collapse toggle produces landscape→square→landscape events and writes `resize:square` into the CSS hook. The aspect ratio that drove the decision does not exist at that moment; a real measurement and an absence of measurement are collapsed into the same answer.

*Symptom:* Orientation-keyed layout flickering through its square branch whenever a panel is hidden and shown.

### 14. `thresholds: [0]` can emit a 'down' crossing that has no matching 'up'  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/intersect.ts:28**

Ascending uses `t > prevRatio` (exclusive) while descending uses `t < prevRatio` with `t >= nextRatio` (inclusive at the bottom). With `lastRatio` initialised to 0, threshold 0 can never fire 'up' but fires 'down' every time the element fully leaves. `[0]` is the natural way to ask "tell me when it enters at all". Related: because `lastRatio` starts at a fabricated 0, an element already 60% visible at mount emits synthetic 'up' crossings for every threshold below 0.6 as if it had just been scrolled in — README recipe 2 would call `loadNextPage()` during mount.

*Symptom:* Paired enter/leave bookkeeping drifting by one, and an infinite-scroll sentinel that loads page 2 before the user scrolls.

### 15. The npm tarball ships only minified dist, while ARCHITECTURE.md tells consumers to take the src folder  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-observe/package.json:27**

`"files": ["dist"]` excludes `src/` and the entry. ARCHITECTURE.md:27 says "Copy-paste consumers: every file under `src/` plus the entry is self-contained TypeScript … take the folder as-is" — the stated primary distribution channel for this repo is the one thing the package does not distribute. The readability work the split exists for is only reachable by finding the GitHub repo, whose URL (`vue-observe`) does not match the package name (`v-observe`) either.

*Symptom:* A consumer who installs then wants to vendor the code gets 11 KB of minified output and has to guess which repo it came from.

### 16. Defensive branches for states the types exclude, one of which would feed a fabricated number into direction inference  ·  `readability`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/intersect.ts:118**

`entry.boundingClientRect?.top ?? 0` optional-chains a non-optional DOM field and substitutes 0 — if it ever fired, `inferDirection` would compare a real previous top against an invented 0 and confidently report 'enter-from-above' or 'leave-to-above'. It exists only because the test factory could have omitted the field (it never does: vObserve.test.ts:331 always supplies a rect). Same class: `if (entry.target !== el) continue` in both observer callbacks (the observers observe one element) and `if (typeof el.matches !== 'function') return false` in src/mutate-records.ts:69. CONVENTIONS bans defensive branches for type-excluded states; each one is a reader's false lead about what can actually happen.

*Symptom:* A reader copying intersect.ts concludes boundingClientRect is unreliable and propagates the ?? 0 into their own measurement code, where it silently produces wrong directions.

### 17. An invalid `match` selector is swallowed per-selector with no development warning  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/mutate-records.ts:74**

`catch { /* Invalid selector — treat as non-match */ }`. With a single selector (the documented form, README recipe 14) one typo means the mode is permanently and silently dead: events are built, filtered to empty, and dropped at src/mutate-records.ts:141. The behaviour is documented as a caveat, which makes it intentional, but there is no `import.meta.env.DEV` warn anywhere in the package, so the consumer's only signal is absence.

*Symptom:* "children:added never fires" with a working subscription, a working handler, and a one-character typo in a selector.

### 18. `ObserveStateAttribute` is public, unused, and its only test is a tautology  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-observe/src/types.ts:186**

Nothing in src/ references the type. `writeStateAttribute` (src/state-attribute.ts:27) builds the string with a raw template literal and returns void, so the type constrains nothing. The only usage anywhere is `const _stateAttr: ObserveStateAttribute = 'intersect:visible;resize:-;mutate:idle'` at vObserve.test.ts:464 — a hand-written string checked against a hand-written type, which asserts that two copies of the same idea agree and cannot fail if the writer drifts from either.

*Symptom:* The grammar changes, the exported type keeps describing the old format, and a consumer's `as ObserveStateAttribute` parser compiles against a contract the runtime no longer honours.

## Comments and docs the code contradicts

- /Users/ozgurseyidoglu/Development/npm/v-observe/src/types.ts:133 — "Gate handler behind `intersect` visibility — wired in a follow-up run." gateOnIntersect is fully implemented and is the package's flagship differentiator with ~25 tests. This text is shipped verbatim in dist/vObserve.d.ts:116, so every consumer's IDE tooltip says the option does nothing yet. (MutateConfig.gateOnIntersect at types.ts:166 has no doc at all.)

- /Users/ozgurseyidoglu/Development/npm/v-observe/src/types.ts:140 — section header "Public types — Mutate (P0 stubs)" over fully implemented, fully tested, fully demoed types.

- /Users/ozgurseyidoglu/Development/npm/v-observe/src/types.ts:46 — "Auto-disconnect after the first `isIntersecting === true` callback." Only true when an `on` callback is also supplied; with crossed-only it never disconnects (verified).

- /Users/ozgurseyidoglu/Development/npm/v-observe/src/intersect.ts:157-159 — "`once: true` ALSO disconnects after the on fire, so subsequent ticks won't fire `crossed` either. That is, `once` collapses everything to a single tick." False for any config without `on`.

- /Users/ozgurseyidoglu/Development/npm/v-observe/src/types.ts:131 — "Coalesces tick storms into one call per window (ms)." It is a trailing-edge debounce that resets on every callback, so a continuous drag produces zero calls until it stops — no call per window. README:325 describes the real behaviour, so the two docs contradict each other.

- /Users/ozgurseyidoglu/Development/npm/v-observe/src/types.ts:129 — "Which ResizeObserverEntry box to read. Default 'border'." Reading is all it does: the observed box is never set, so what the option cannot do (change when callbacks fire) is the half consumers need for device-pixel.

- /Users/ozgurseyidoglu/Development/npm/v-observe/README.md:562 — "`crossed` events also stop — `once` is the explicit collapse signal." Verified false without `on`.

- /Users/ozgurseyidoglu/Development/npm/v-observe/README.md:564 — "If a required global is undefined at mount, that mode short-circuits — the state attribute still writes its initial sentinel." The intersect "sentinel" in that case is `hidden`, not `-`; the suite itself calls `hidden` the false positive at vObserve.test.ts:1106.

- /Users/ozgurseyidoglu/Development/npm/v-observe/README.md:138 — recipe "Nested scroll container — custom `root`" passes a template ref inside the binding expression; `root` is null at mounted and the real element is never applied (verified with real Vue).

- /Users/ozgurseyidoglu/Development/npm/v-observe/README.md:482 — "from: ResizeOrientation | null // null until the second orientation". No orientation event is emitted until the second orientation, so `from` is never null.

- /Users/ozgurseyidoglu/Development/npm/v-observe/README.md:477 — "bracket: string // bracket label entered". On a multi-bracket jump every event carries the final label, not the bracket entered at that threshold (verified: [320,'>=640'],[640,'>=640']).

- /Users/ozgurseyidoglu/Development/npm/v-observe/README.md:284 — "`box: 'device-pixel'` falls back to contentBox × devicePixelRatio when the browser lacks devicePixelContentBoxSize." True as far as it goes, and it conceals that no callback is delivered on a DPR change at all, which is the only reason to choose this box.

- /Users/ozgurseyidoglu/Development/npm/v-observe/ARCHITECTURE.md:27 — "Copy-paste consumers: every file under `src/` plus the entry … take the folder as-is", contradicted by package.json:27 `"files": ["dist"]`.

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-observe/manifest.ts:10 — "the intersect demos use their own root so the results are reproducible." All seven demos that pass `root: scroller` observe the viewport; the stated reason the demos are reproducible is the thing that does not happen.

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-observe/05-root-margin.vue:50 — "`root` is the scroll box, not the viewport, so this stays reproducible regardless of where the page is scrolled." Exactly inverted, and the rootMargin slider above it is inert because the observer is never rebuilt.

## ARCHITECTURE.md claims nothing enforces

- "Each mode owns exactly one segment; writing one never clobbers the others" (src/state-attribute.ts:2-4, ARCHITECTURE.md:23-25) — enforced by nothing. `segments` is a plain mutable object handed to every mode, and segments are in fact written from six places across four files (intersect.ts:136, resize.ts:214/242, mutate.ts:60/68/165, directive.ts:50/62/69), including directive.ts writing all three. Nothing stops resize from setting `segments.intersect`, and a reader of state-attribute.ts sees only the initial values and the writer, not where segments actually change.

- "Mode isolation: intersect.ts, resize.ts and mutate.ts never import each other" (ARCHITECTURE.md:21) — currently true, enforced by nothing: no lint rule, no dependency check, no test. More to the point, isolation is already honoured in letter only: gate.ts:33-51 reaches into resize's and mutate's private fields (lastDispatched, lastOrientation, timer, pending) and hand-maintains the reset list. The real invariant — "a gated mode's baseline is fully cleared on restore" — lives in a different module from the state it clears, with no link back, so adding any new baseline field to ResizeInternal silently breaks it.

- "dependencies point strictly downward — no cycles" (ARCHITECTURE.md:2) — true today, checked by nothing. tsup would happily bundle a cycle.

- "Each module has one purpose" (ARCHITECTURE.md:2) vs directive.ts, described as "lifecycle wiring only", which duplicates both setup functions' cfg-swap logic and writes every mode's state segment.

- The ObserveStateAttribute template-literal type (src/types.ts:186) is presented as the grammar of the attribute, but writeStateAttribute (src/state-attribute.ts:27) is typed `void` and builds the string by hand — the type cannot constrain the writer, and no test derives one from the other.

- gate.ts's two stated fallbacks are inconsistent about failure direction and nothing reconciles them: the gate fails OPEN when intersect is unwired ("graceful degradation prefers always fire"), while the CSS hook for the same unwired intersect fails CLOSED at `intersect:hidden`. Whichever is right, both cannot be.

## Tests passing for the wrong reason

- vObserve.test.ts:3440 'once: true visible disconnect → gated resize remains un-gated after restore (intersect stays "visible")' — the inline comment claims "lastIsIntersecting is `true` in the internal, so subsequent gated dispatches pass through". There is no internal: teardownIntersect deleted it, so the test passes through gate.ts's `if (!i) return false` branch, i.e. the "IntersectionObserver unavailable" degradation path. The test asserts a permanently-broken gate as intended behaviour and documents a mechanism that does not exist.

- vObserve.test.ts:2868 'reactive `box` swap takes effect on the next tick (box read at callback time)' — passes only because MockResizeObserver.observe(el) (vObserve.test.ts:120) accepts no init argument, so the missing `{ box }` on the real observe() call is invisible. The test enshrines the read-side half of a two-sided option as if it were the whole feature.

- vObserve.test.ts:3121 'resets resize baseline on hidden→visible transition (first post-restore tick has from:null)' and the rest of the gate suite — every post-restore assertion is driven by a hand-fired resize callback. A real ResizeObserver does not re-deliver the observation it lost while gated, so the suite proves the baseline reset given an event the browser will not produce.

- vObserve.test.ts:464 'public type exports are importable (compile-time presence check)' — `const _stateAttr: ObserveStateAttribute = 'intersect:visible;resize:-;mutate:idle'` compares a literal typed by hand against a type written by hand. It cannot fail if writeStateAttribute drifts from either, which is the only thing worth checking.

- vObserve.test.ts:1975 '"attr:*" subscribes to all attributes (no attributeFilter)' — asserts the init shape only. Because every MutationObserver in the suite is fired manually and never observes a real DOM write, the test certifies the exact configuration that freezes a real browser (confirmed with a real MutationObserver).

- vObserve.test.ts:1935 — test titled 'throws when mutate config is provided but handler is missing — no, missing handler is a no-op': the title argues with itself, which means nobody re-read it after the expectation was inverted.

- vObserve.test.ts:598 'forwards thresholds to IntersectionObserver options' — correct for mount, and it is the only threshold-plumbing test, so nothing covers the fact that a thresholds swap changes the crossed math without changing what the observer samples.
