# Quality audit — `v-fit-children`

**Verdict: structurally-unsound**  ·  17 findings

**PUBLISHED ON NPM — defects here are live for real consumers.**


## The single worst thing

The package's only output — the `fit-children-updated` event — is gated on a signature that does not include what the event reports. `applyFit` suppresses the dispatch when `sameRun(fit.visible, state.visible) && state.data === state.lastDispatchedData`, i.e. on the *visible* set plus the array's *reference identity*; the payload is entirely about the *hidden* set. Add or remove hidden children by mutating the array in place — `items.push` / `pop` / `splice`, the dominant Vue idiom and exactly what playground demo 06's buttons do — and neither half of the gate moves, so no event fires and the consumer's "+N more" badge keeps reporting a world that no longer exists. Measured against `dist/vFitChildren.min.js`: 6 chips in 300px, then three hidden ones popped off the array and the DOM. Reality: 5→3 children, 0 hidden. Last event still says `hiddenChildrenCount: 3`, `hiddenData: ["d","e","f"]`, `hiddenIndices: [3,4,5]` — and `hiddenChildren` holds three detached elements. In the same pass `data-v-fit-state` is correctly written to `fits`, because the attribute write sits *above* the early return. So the two outputs of one pass contradict each other, the wrong one is the one consumers render from, and nobody debugging a stale badge would look in the dispatch guard for a bug about hidden children.


## Findings

### 1. The dispatch gate watches the visible set and the array's reference; the event reports the hidden set  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/visibility.ts:70**

`unchanged = sameRun(fit.visible, state.visible) && state.data === state.lastDispatchedData`. Neither term changes when hidden children are added or removed, or when the array is mutated in place — which is how Vue code mutates arrays. The comment above it claims the signature 'covers the visible set AND the data reference, because an unchanged fit over new data still changes hiddenData'; it covers neither of the two ways hiddenData actually changes. The 2.2.0 changelog entry 'Swapping the data array with an unchanged fit left hiddenData stale' fixed only the immutable-replacement half.

*Symptom:* A '+3 more' badge, and the dropdown behind it, listing rows the user already deleted. Probed against the built bundle: after popping three hidden items, reality is 0 hidden children while the last event still reports 3 and names the deleted items; the host's data-v-fit-state correctly reads 'fits' at the same moment. Live in playground demo 06 — click Add until chips are hidden, then Remove child, and the pre block freezes.

### 2. data-v-fit-keep consumes a data index, keepVisibleEl does not — so hiddenData silently shifts  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/dom.ts:79**

`isDataChild` excludes DECORATIVE_ATTR and the keepVisibleEl child, but never KEEP_ATTR. The README presents the two pinning mechanisms as equivalent ('Both methods can be used together'), and `v-fit-children` is documented to pin either way. Worse, `visibility.ts:80`'s `.filter((index) => index < state.data!.length)` swallows the resulting out-of-range index, so the skew surfaces as a quietly short array rather than an error — a defensive branch that converts a mapping bug into a wrong answer, against CONVENTIONS' 'no defensive branches'.

*Symptom:* Probed against the built bundle with a pinned child first and four data chips: data-v-fit-keep gives hiddenData ["delta"] for two hidden chips; the identical layout with keepVisibleEl gives ["charlie","delta"]. So hiddenChildrenCount is 2 while hiddenData.length is 1, and the one name shown is the wrong one. No test and no demo combines `data` with either pin mechanism.

### 3. Children are measured at their post-constraint width, so the fit decision is fed the answer it is trying to compute  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/measure.ts:59**

`width = overflowX !== 'visible' ? rect.width : Math.max(rect.width, child.scrollWidth)`. In a flex host — the shape every demo and the README's inline-badge recipe use — children carry the default `flex-shrink: 1`, so `rect.width` is what the host *allowed*, not what the child *wants*. A chip with `overflow: hidden` or `text-overflow: ellipsis` (the single most common chip CSS there is) takes the first branch and its shrunken width; a chip with wrappable text takes the second, but its scrollWidth equals the shrunken box because the text wrapped instead of spilling. Either way sum(cost) collapses to the available width by construction, `full <= available + EPSILON` is satisfied, and the directive concludes 'fits'. Every demo escapes this only because the playground's shared `.pg-chip` rule sets `white-space: nowrap`, which raises the child's automatic minimum size to its full width. Nothing in the README's Known limitations mentions that children must resist shrinking, and the scrollWidth read is advertised as a bonus feature ('Accounts for content overflow') rather than the load-bearing crutch it is.

*Symptom:* A consumer copies the inline-badge recipe, puts `overflow: hidden; text-overflow: ellipsis` on their chips, and the directive silently does nothing at all: every chip stays visible, squashed to a few ellipsised pixels each, data-v-fit-state reads 'fits', and no event ever fires. The playground cannot reveal it because its chips are nowrap.

### 4. state.hostWidth holds two different quantities depending on which module wrote it last  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/observers.ts:30**

`measure.ts:31` writes `getContentWidth(host)` = border-box rect minus border and padding, taken with every child un-hidden. `observers.ts:30` writes `entry.contentRect.width`, taken with children hidden. Those are not the same number: getBoundingClientRect's border box includes a classic scrollbar, contentRect excludes it, so on Windows/Linux the two disagree by the scrollbar width permanently, and the discrepancy flows straight into `available = Math.min(state.hostWidth, state.containerWidth)` and into the two change-detectors that compare the two lineages against each other (`available > state.lastAvailable + EPSILON` in observers.ts:82, `Math.abs(width - state.parentWidth) > EPSILON` at observers.ts:46, `Math.abs(width - state.containerWidth) > EPSILON` at observers.ts:33). On macOS overlay scrollbars the gap is zero — it is invisible on the author's machine by construction.

*Symptom:* In a scrolling container on Windows: the budget is ~15px too generous, so one chip too many is admitted and sits clipped at the host's edge — the exact symptom the repo already burned Run 19 on. Separately, the container/parent 'did the geometry change' tests are then true on every single ResizeObserver delivery, so `state.oversizedRuns` (the feedback-loop record) is wiped continuously and the loop guard effectively does not exist.

### 5. The 'shrinking is free, no DOM reads' fast path cannot produce an answer, only confirm or redo  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/schedule.ts:40**

`if (!remeasure && !sameRun(fit.visible, state.visible)) { runPass(state, true); return }` — the cached pass's only two outcomes are 'nothing changed' or 'discard everything and re-measure'. It can never apply a different run. So the advertised optimisation is a speculative arithmetic pre-pass that is pure overhead in exactly the case the docs promise savings for (a shrink that drops a chip), and is a tautology otherwise: for a content-sized host `available` equals the sum of the currently visible children, so the greedy walk re-admits precisely the set it was given. The claim is repeated in four places: README 'Shrinking is free … no DOM is read at all', ARCHITECTURE.md's third invariant, the playground manifest's fourth note, and the brief. Roughly 30 lines of state (`lastAvailable`, `appliedAvailable`, the `remeasure` plumbing) and 40 lines of comment exist to maintain it.

*Symptom:* A maintainer optimising a slow resize trusts the docs, looks past `remeasure`, and never discovers that every chip-dropping resize runs the decision twice. A copy-paste consumer simplifying 'the redundant branch' either loses the only thing preventing the host's post-hide width from feeding back (collapse to zero) or deletes a path that was never load-bearing — and the code gives no way to tell which.

### 6. ARCHITECTURE.md's central invariant is violated by three of the four modules it names  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/ARCHITECTURE.md:24**

'measure.ts reads, visibility.ts writes, fit.ts does neither.' measure.ts's first act is `children.forEach((child) => showChild(child))` — the most consequential DOM write in the package, mutating what the user sees mid-pass. visibility.ts reads `host.children`, `child.style.display` and `host.getAttribute`. schedule.ts writes, via `ensureHideRule` injecting a stylesheet. And observers.ts is a second, independent measurement source — it writes hostWidth/containerWidth/parentWidth from ResizeObserver contentRects taken in a different DOM state — which is precisely 'a second, disagreeing measurement', the defect class the invariant exists to forbid. measure.ts:3 defends the letter of it ('Nothing else in the package calls getBoundingClientRect') while the substance leaks in through a different API.

*Symptom:* Nothing enforces any of it: `FitChildrenState` is a fully mutable record handed to every module with no readonly fields, there is no lint config, no typecheck script, and no test asserting the separation. The next person to 'just read the width here' does so with the document's blessing and reintroduces the 2.x partial-visibility bug, exactly as the document claims is impossible.

### 7. widthRestrictingContainer is inert in the usage the README teaches, and the README describes arithmetic the code does not perform  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/measure.ts:35**

`available = min(host, container)`. When the container is an ancestor — the documented case, the Quick start's `widthRestrictingContainer: containerRef`, and playground demo 04 — the host is always the narrower box, so the option changes nothing whatsoever. README:90 says 'When they differ, the directive element's own margin, border, and padding are subtracted from the available space': no subtraction happens in either direction, and if the host ever were the wider box the container's value would be used raw, with the host's own padding and border still uncounted. Zero tests touch the option; demo 04 exists to demonstrate it and demonstrates a no-op while its comment claims the host's border, padding, margin 'and the +N badge beside the row are all accounted for' — they are accounted for by the `min`, whatever you pass.

*Symptom:* A consumer debugging a row that drops one chip too many adds `widthRestrictingContainer`, sees no change, and concludes the directive is broken in some other way. The one configuration where the option matters (a host wider than its container) is the one nobody documents, tests or demos.

### 8. A dead branch preserved by a comment that cites a test which does not exist  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/measure.ts:58**

'The `||` fallback is load-bearing: the gap tests mock getComputedStyle down to two keys.' grep for getComputedStyle in vFitChildren.test.ts: zero hits. No test mocks it, and the 'gap tests' now read spacing from laid-out positions rather than computed gap, so the mock could not exist. The fallback is also unreachable on its own terms: `getComputedStyle(child).overflowX` is only '' in environments where `child.style.overflowX` is '' too, so the `||` can never change the result. instructions/v-fit-children.md repeats the same claim as a 'gotcha — verify before cleaning up', which means the false justification is now load-bearing for the next maintainer rather than for the code.

*Symptom:* A reader of the source — the product — is told a branch is required by a test harness that is not there, in the one function whose correctness the whole package rests on. The same paragraph is what would stop them deleting it.

### 9. `previousRight` advances by the spill width, so every gap after a spilling child is silently dropped  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/measure.ts:72**

`previousRight = rect.left + width`, where `width` may be `scrollWidth` (larger than the box). The next child's real `rect.left` is based on the previous child's *box*, so `Math.max(rect.left - previousRight, 0)` clamps to 0 and the spacing in front of it vanishes from the budget. The feature that handles overflow: visible children therefore deletes the gap after each of them.

*Symptom:* A row of content-spilling children is billed for n−k gaps instead of n−1 and admits a chip too many, which then renders clipped at the host edge — attributed to the gap option or to CSS, never to the one line that advanced a cursor by the wrong quantity.

### 10. The test guarding the package's worst historical regression is insensitive to the input it claims to feed  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/vFitChildren.test.ts:526**

'Feed the post-hide width back in the way a real ResizeObserver does.' It does not: `MockResizeObserver.trigger` reports `hostWidth` for the host, a module-level `let` that `buildShrinkToFit` never assigns and `beforeEach` never resets — so the five iterations feed a constant left over from whichever buildRow test ran last. Run the describe in isolation (`vitest run -t "a host that sizes to its own children"`) and the fed value is 0 for all five iterations; it still passes, because the re-measure guard rescues any wrong input. The assertion therefore cannot distinguish a working feedback guard from no feedback at all.

*Symptom:* The one behaviour this package has regressed on twice — a content-sized host walking 3 visible to 0 — is covered by a test that would stay green if the guard were deleted and the observer fed nothing. The suite's own header comment about a no-op observe() keeping the suite green 'while the per-child observer did not exist at all' describes this test precisely.

### 11. The loop-breaking record is wiped by the render most likely to need it  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/directive.ts:114**

`updated` clears `state.oversizedRuns` on any option change or any sibling-identity change. A `v-if` '+N' badge appearing or leaving IS a sibling-identity change, and that badge is the documented cause of the oscillation the record exists to break (README Known limitations, last bullet; the 2.2.0 changelog's 'A badge labelled from the hidden set could cycle forever'). observers.ts is careful to clear it only when `!state.lastApplyMoved`; the directive hook throws that care away unconditionally. The record also retains detached elements in its four slots after a v-for removal, where they can never match again but still occupy capacity.

*Symptom:* The guard is armed for label-only badge changes (same element) and disarmed for `v-if` badges — the form the README's own Quick start uses. A non-monotonic badge in that shape can flip-flop across frames with the record永 empty, which is the 'cycles 5 -> 6 -> 7 -> 5 forever at ~60 rounds a second' the schedule.ts comment says is handled.

### 12. `remeasure` is assigned rather than latched in the observer loop, making growth detection depend on entry delivery order  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/observers.ts:50**

`remeasure = width > state.parentWidth + EPSILON` overwrites whatever a previous entry in the same callback decided; every other branch latches it to true. It happens to work only because the `observe()` calls at lines 85–107 register host, container, parent, siblings, children in that order and ResizeObserver delivers in observation order, so the clobbering branch runs before the latching ones. Nothing in the file says so.

*Symptom:* Reordering the observe() calls for readability — or a browser that batches entries differently — silently turns a growth signal into a cached pass, and the row stays collapsed at whatever the narrowest moment produced. The symptom appears on resize-out only, looks like a missed event, and the cause is an `=` where the neighbours use a latch.

### 13. README contradicts itself on isOverflowing, the flagship boolean  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/README.md:181**

The event table says 'true if any children were hidden, false if all fit'. fit.ts:79 returns true whenever the content exceeds the available width regardless of whether anything could be hidden, the changelog at README:375 lists that as a deliberate fix, and a test asserts it for an all-pinned row. README:183 adds a second false claim — 'When all children fit (including the offset), isOverflowing is false' — when the smart-fit branch at fit.ts:46 tests `full <= available + EPSILON` with the offset deliberately excluded, as its own test ('does not reserve the offset when everything fits') asserts.

*Symptom:* A consumer writes `v-if="isOverflowing"` on a '+N more' badge and gets a badge reading '+0 more' on an all-pinned clipped row, or reasons about the offset from a sentence that inverts the condition.

### 14. README tells you to substitute hiddenIndices for hiddenData, across two different index spaces  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/README.md:272**

'hiddenIndices is always provided regardless of the data option, so you can also map manually if needed.' hiddenIndices are DOM child indices (visibility.ts:55 pushes the forEach index, which counts decorative children and consumer-hidden children); hiddenData indices are a separate counter that skips both. The two coincide only in a row with no separators and no v-show. Demo 02 prints them side by side in exactly such a row, teaching that they are interchangeable; demo 08, which has separators, prints hiddenData and quietly omits hiddenIndices.

*Symptom:* `myArray[hiddenIndices[0]]` names the wrong item, or is undefined, the moment the consumer adds a separator or a v-show child — and the README is where they got the idea.

### 15. The package brief and CLAUDE.md describe an engine that is not in the repository  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/instructions/v-fit-children.md:19**

The brief's 'TypeScript exports (the 2.2.0 contract)' lists `FitChildrenPlugin` and `DIRECTIVE_NAME`; directive.ts:139 explicitly documents their absence ('No plugin export, and no registration name constant'). CLAUDE.md repeats the claim and adds a third fiction, `data-fit-children-state`, where the code stamps `data-v-fit-state`. The brief's '3.0 engine' section describes per-recalculation rAF (schedule.ts opens by saying there is none), visibility.ts sizing the host with width/max-width/flex (it never touches the host's size), measure.ts releasing imposed widths (it imposes none), and 'beforeMount (not mounted) is safe here because all measurement is rAF-deferred and the !state.targetElement && wrapperElement dance compensates' — no wrapperElement exists and the hook is `mounted`. Its 'No width caching, ever … every pass re-measures fresh' is contradicted by `state.metrics` plus the entire `remeasure` design. Four version numbers are live at once: package.json 2.2.0, README changelog 2.2.0, brief '3.0.0 built locally pending publish', and a stray ozjsey-v-fit-children-3.0.0-rc.4.tgz in the package root.

*Symptom:* The document the repo designates as 'the source of truth for intent' sends the next session (or the next contributor) to preserve invariants for code that does not exist and to import exports that do not exist. The first thing anyone does with a brief is trust it.

### 16. Unused import and unused parameter in the product source, with nothing in the repo able to notice  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-fit-children/src/observers.ts:7**

`import { getContentWidth } from './dom'` is never called in observers.ts — a fossil from when the observers read boxes themselves, which is still what ARCHITECTURE.md:11 says they do ('every trigger: container / host / parent / child boxes'). `getContentWidth`'s second parameter (`style = window.getComputedStyle(element)`) is never supplied by any caller. tsconfig sets `strict` but not `noUnusedLocals`/`noUnusedParameters`, there is no lint config, and package.json has no lint or typecheck script — build is tsup, which does not care.

*Symptom:* For a package whose stated distribution model is 'people copy the folder', a dead import in the module map's own terms is the first thing a reader trips on, and it proves no automated check reads these files at all.

### 17. Playground demo 05 describes demo 01's code incorrectly; demo 06 credits the wrong mechanism  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-fit-children/05-inline-badge.vue:43**

Demo 05: 'Compare with demo 1, where the badge sits below the row and offsetNeededInPx holds width back inside it anyway.' Demo 01's frame is `display: flex`, the badge is an inline flex sibling, and the binding is `{ offsetNeededInPx: 0 }` — every clause is false, and the two demos in fact differ only by `flex: 1` on the host. Demo 06's caption credits the per-child ResizeObserver for 'Grow the first child', but that button assigns `items[0] = items[0] + ' (longer now)'` and `:key="item"`, so Vue replaces the element and `childrenChanged()` catches it through the `updated` hook. No demo exercises the per-child ResizeObserver at all — the mechanism the brief says was once entirely missing while the suite stayed green.

*Symptom:* These files are copy-paste examples; the prose is the only explanation a stranger gets. One sends them looking for a reserved-offset behaviour that is set to 0, the other assures them a trigger is covered in the playground when it is covered nowhere.

## Comments and docs the code contradicts

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/src/measure.ts:58 — "The `||` fallback is load-bearing: the gap tests mock getComputedStyle down to two keys." No test in the suite mocks getComputedStyle (zero grep hits), and the fallback is unreachable regardless.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/src/measure.ts:2 — "The one layout read per pass. Nothing else in the package calls getBoundingClientRect, which is what keeps a second, disagreeing measurement from existing." Literally true about that one API; observers.ts writes hostWidth/containerWidth/parentWidth from ResizeObserver contentRects, in a different DOM state, into the same decision.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/src/measure.ts:11 — "The one layout read" / "one rect pass over real children": the function performs four element reads on the host/parent/container (two separate getBoundingClientRect calls on the host) plus a rect, a getComputedStyle and a scrollWidth per child.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/src/visibility.ts:67 — "The signature covers the visible set AND the data reference, because an unchanged fit over new data still changes hiddenData." It covers neither of the two real ways hiddenData changes: in-place array mutation, and hidden children being added or removed.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/src/constants.ts:17 — "Layout resolves to 1/64px, so an exact <= can reject a run that fits by 0.002px." 1/64 is 0.015625; 0.002 is not a layout quantum.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/ARCHITECTURE.md:24 — "measure.ts reads, visibility.ts writes, fit.ts does neither." measure.ts un-hides every child, visibility.ts reads children/style/attributes, schedule.ts injects a stylesheet.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/ARCHITECTURE.md:11 — observers.ts described as reading "container / host / parent / child boxes"; it reads no boxes, and the leftover getContentWidth import is the fossil.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/ARCHITECTURE.md:36 and README.md:307 — "Shrinking is free … the pass costs no DOM reads at all." A shrink that changes anything re-measures; the cached pass can only confirm or be discarded.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/README.md:90 — "the directive element's own margin, border, and padding are subtracted from the available space." The code takes min(host, container); nothing is subtracted.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/README.md:181 — "isOverflowing: true if any children were hidden" — contradicted by fit.ts:79, by the changelog at README:375, and by a test.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/README.md:183 — "When all children fit (including the offset), isOverflowing is false" — smart fit tests the total without the offset.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/README.md:272 — "hiddenIndices … so you can also map manually if needed" — DOM indices and data indices are different index spaces.

- /Users/ozgurseyidoglu/Development/npm/v-fit-children/README.md:229 — "Both methods can be used together" — data-v-fit-keep and keepVisibleEl produce different hiddenData for the same layout.

- /Users/ozgurseyidoglu/Development/npm/instructions/v-fit-children.md:19 — FitChildrenPlugin and DIRECTIVE_NAME listed as "the 2.2.0 contract"; directive.ts:139 documents their deliberate absence.

- /Users/ozgurseyidoglu/Development/npm/instructions/v-fit-children.md — "inside one rAF", "visibility.ts … sizes the host to exactly that run", "Sizing the host needs all three of width, max-width and flex", "beforeMount (not mounted) is safe here because all measurement is rAF-deferred and the !state.targetElement && wrapperElement dance compensates", "No width caching, ever … every pass re-measures fresh": none of this matches the shipped code.

- /Users/ozgurseyidoglu/Development/npm/CLAUDE.md — claims plugin parity (FitChildrenPlugin + DIRECTIVE_NAME) and a data-fit-children-state attribute; neither exists (the attribute is data-v-fit-state).

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-fit-children/05-inline-badge.vue:43 — describes demo 01 as having the badge below the row with offsetNeededInPx holding width back; demo 01 is an inline flex badge with offsetNeededInPx: 0.

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-fit-children/06-dynamic-children.vue:50 — credits the per-child ResizeObserver for a change that alters the v-for :key and is therefore handled by the updated hook.

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-fit-children/04-gap-and-container.vue:31 — claims naming an ancestor as widthRestrictingContainer accounts for the host's border/padding/margin and the badge; min(host, container) makes the option inert there.

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-fit-children/manifest.ts — "One ResizeObserver watches the container and every child" (it also watches the parent and every sibling, which the README calls the headline feature) and "shrinking it is pure arithmetic … costs no DOM reads at all".

## ARCHITECTURE.md claims nothing enforces

- "measure.ts reads, visibility.ts writes, fit.ts does neither" (ARCHITECTURE.md:24) — violated today by measure.ts (un-hides every child), visibility.ts (reads children, style.display, getAttribute) and schedule.ts (injects a stylesheet). Nothing enforces it: FitChildrenState is a mutable record with no readonly fields, handed to every module; no lint config exists; package.json has no lint or typecheck script; no test asserts the separation.

- "One measurement per pass, used by one decision, applied by one writer. A second, disagreeing measurement is the defect class that produced partially-visible children for the whole of 2.x" (ARCHITECTURE.md:24-27) — observers.ts is exactly that second source: it writes hostWidth, containerWidth and parentWidth from ResizeObserver contentRects taken with children hidden, using a different geometry definition than measure.ts's getContentWidth, and the fit decision consumes whichever wrote last.

- "Hiding never touches el.style.display" (ARCHITECTURE.md:29) — this one is actually true in the code, but the only thing preventing a regression is prose. isConsumerHidden's correctness depends on it and would silently invert if anyone ever wrote display from dom.ts; no test or type would notice.

- "A cached pass may only confirm the current run, never change it" (ARCHITECTURE.md:36) — enforced by one `if` in schedule.ts with no test behind it (see the shrink-to-fit tests above, which pass regardless), and undermined by the fact that the cached pass can never produce a different answer anyway.

- "Its width is never used as a budget" about the parent (ARCHITECTURE.md:40) — true, but state.parentWidth sits in the same flat mutable state object next to hostWidth and containerWidth, with identical types and no naming or structural barrier; the only thing stopping a future `min(..., state.parentWidth)` is the comment.

- "every file under src/ plus the entry is self-contained TypeScript with no dependencies beyond the vue peer — take the folder as-is" (ARCHITECTURE.md:43) — true of imports, but nothing verifies the folder actually compiles or lints standalone: tsconfig omits noUnusedLocals, there is no typecheck script, and an unused import already shipped (observers.ts:7).

- instructions/v-fit-children.md: "No width caching, ever. Commit 984a2ef removed it deliberately; every pass re-measures fresh." — state.metrics is a per-child width cache and the remeasure flag exists specifically to decide when not to re-measure. The stated rule and the shipped design are opposites, and nothing reconciles them.

## Tests passing for the wrong reason

- vFitChildren.test.ts:520 "settles instead of collapsing to zero when an offset is reserved" and :535 "settles with no offset too" — the comment says the loop feeds the post-hide width back; trigger() actually reports the module-level `hostWidth`, which buildShrinkToFit never assigns and beforeEach never resets, so all five iterations feed a constant leaked from an earlier describe. Run in isolation it feeds 0 and still passes, because the re-measure guard neutralises any value. The test cannot fail if the feedback handling is removed.

- vFitChildren.test.ts:294 "puts the hide rule back if something removed it" — it also changes hostWidth from 300 to 150 in the same step, so the pass it relies on is a resize pass; it never exercises the case the comment is about (a pass where nothing else changed), which would return early from runPass without... in fact ensureHideRule runs first in recalculate, so the assertion holds, but the width change is unexplained noise that hides which code path is under test.

- vFitChildren.test.ts:701 "does not let decorative children consume a data index" — asserts hiddenData only. hiddenIndices in that same row are DOM indices that do NOT line up with the data array, which is the failure mode the README then tells consumers to rely on. The test's scope stops one assertion short of the bug.

- vFitChildren.test.ts:680 and :278 — every data-mapping test puts the non-data children last or uses no pins, so the data-index skew from data-v-fit-keep (confirmed by probe) is outside the suite's reach. No test combines `data` with either pin mechanism, and none combines `data` with v-show.

- The whole suite: `widthRestrictingContainer` is never passed to any test, so the containerWidth half of `available = min(host, container)` — the package's flagship option — has zero coverage. Likewise nothing ever delivers a ResizeObserver entry for the host's parent, so the parent-as-growth-signal branch (observers.ts:38-53) is untested, and nothing exercises oversizedRuns / provenTooBig (schedule.ts:66-86), the most intricate 20 lines in the package.

- vFitChildren.test.ts:465 "does not re-enter while a pass is running" asserts passes === 1 — i.e. it pins the behaviour that a recalculation triggered from inside the event dispatch is dropped and never rescheduled, framed as a feature.
