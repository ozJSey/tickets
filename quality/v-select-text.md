# Quality audit — `v-select-text`

**Verdict: needs-work**  ·  15 findings


## The single worst thing

The package's flagship guarantee — "a host that mounts empty stays armed and selects its text on the render that brings the text in" (README → *The empty host*, all 29 tests in vSelectText.empty.test.ts, playground card 13, the entire reason `edgeSpentMap` stores "a selection landed" instead of "enabled") — is driven by the wrong input. The only thing that can re-run a cycle is the directive's `updated` hook, which fires when the *owning component* re-renders, not when the host's text changes. I mounted `<p v-select-text><Child/></p>` where the child owns the reactive text: the text arrives, `el.textContent` is correct, the edge is still unspent, and **zero `select-text` events fire, ever**. The same holds for a slot filled by a parent, a `v-html` written by anything else, or any DOM mutation outside that component's render. Every proof of this promise manufactures the trigger: `empty.test.ts` calls `updated(el, true)` by hand, and card 13 interpolates `{{ fromApi }}` in the same component — the one shape where the proxy happens to hold. Four repair passes and three behaviour audits all asked "does the edge fire when the text lands", and the defect is one layer below, in what decides that a cycle should run at all.


## Findings

### 1. Late text only lands if the owning component happens to re-render  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/directive.ts:96**

`updated` is the package's only re-entry point for the render-driven triggers, and Vue runs it when the component that carries the directive re-renders — never because the host's text changed. The whole SEL-4 design (edgeSpentMap storing "a selection landed", not "enabled") exists to keep the host armed until the text arrives, but nothing ever asks again unless the owning component re-renders. There is no MutationObserver and no re-check. The decision 'should I run a cycle now?' is fed by 'my owner re-rendered' as a proxy for 'my text changed', and the proxy only holds when the text is interpolated in the same component.

*Symptom:* Verified with a real app: `<p v-select-text><Child/></p>` where Child owns the text — after the text arrives the host's textContent is correct, the edge is unspent, and 0 select-text events fire. Same paragraph with the text interpolated locally fires 1. A consumer files "v-select-text doesn't work with async content", the maintainer reproduces it with the local-interpolation shape, it works, and the bug is closed as unreproducible.

### 2. whitespace:'preserve' selects — and copies — text nobody can see, silently  ·  `leak`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/text-map.ts:145**

The 'preserve' branch is a bare TreeWalker over every text node: it does not skip `display:none`, `<script>`, `<style>`, `<template>` or anything else the collapse walk filters. So `match` searches hidden text, `anchorAt` returns anchors inside hidden nodes, and `copy: true` writes whatever it found. Neither diagnostic catches it: `warnIfUnselectable` and `warnIfNotRendered` inspect the *host*, not the nodes the Range actually covers, so a visible host with a hidden child produces a confident selection over invisible text with zero warnings. README:284 states the invariant flatly — "A match can never reach into text the user cannot see" — and README:294 contradicts it twenty lines later with "'preserve' is raw textContent, verbatim — no skipping". The manifest repeats the absolute version: "what you cannot see never reaches the clipboard."

*Symptom:* Verified: `<p v-select-text="{ whitespace: 'preserve', match: 'SECRET-TOKEN' }">` over `<span style="display:none">SECRET-TOKEN</span>visible` reports `text: "SECRET-TOKEN"`, fires 0 warnings, and with `copy: true` puts it on the clipboard. A code-block host (the documented use for 'preserve') with a hidden draft/internal fragment leaks it on a click-to-copy.

### 3. A non-string, non-RegExp `match` throws out of `mounted` — and out of every click  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/find-range.ts:46**

`findOccurrences` branches on `typeof match === 'string'` and treats everything else as a RegExp, reaching `match.flags.replace(...)`. `resolve.ts:29` already added a defensive branch for exactly this crash family, with the rationale "a template is not typechecked at runtime" — but it guards only the outer binding value and passes `value.match` straight through unvalidated. The likely route is unguarded and the unlikely one is guarded. With `trigger: 'click'` the throw escapes a raw `addEventListener` handler, so there is no Vue error boundary at all.

*Symptom:* Verified: `{ match: 0 }`, `{ match: false }`, `{ match: {} }`, `{ match: new Date() }` all throw `TypeError: Cannot read properties of undefined (reading 'replace')` from inside the lifecycle hook. `<code v-select-text="{ trigger: 'click', match: orderId }">` where `orderId` is a number from an API response logs an uncaught TypeError on every single click, with a stack pointing at a minified `dist` bundle.

### 4. An offset the code cannot use means "select the entire host", not "select nothing"  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/find-range.ts:75**

`normalizeOffset` returns `undefined` for NaN, ±Infinity and non-numbers, and `findRange` reads "both undefined" as `mode: 'all'`. So an unusable `start`/`end` collapses into the same meaning as "no range given", which is the maximally destructive reading of an invalid request. types.ts:89 documents the opposite ("`>` the text length is clamped to the text length"), which would make the request collapsed and therefore a no-op; README:234 says "`NaN` / `Infinity` are treated as unset" without saying that unset means everything.

*Symptom:* Verified: `{ start: Infinity }` on a 19-char paragraph reports `start: 0, end: 19` and selects the whole host. `{ start: total / count }` with `count === 0`, or `{ start: Number(field.value) }` on an empty field, silently turns "highlight one word" into "highlight the paragraph" — and with `copy: true`, the whole paragraph onto the clipboard.

### 5. useSelectText().clear() wipes a selection this host never made  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/use-select-text.ts:150**

`clearApi` calls `sel.removeAllRanges()` unconditionally, with no check that the document selection belongs to this target. This is the exact hazard ARCHITECTURE.md:70-77 was written about — "`applyRange` opens with `removeAllRanges()`, so a host that resolved to nothing would otherwise wipe whatever the user had selected" — enforced in selection.ts and then reintroduced in the sibling file, where nothing prevents it. ARCHITECTURE's "selection.ts is the only place a selection happens" is also untrue here: `clearApi` calls `setSelectionRange(0, 0)` on inputs directly.

*Symptom:* Verified: user selects text in an unrelated `<aside>`, code calls `api.clear()` on a `<p>` that never selected anything, and the user's selection is gone. A route-leave hook or a modal-close handler that tidily calls `clear()` destroys whatever the user had highlighted elsewhere on the page.

### 6. copyState has no sequence guard, unlike the attribute it mirrors  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/use-select-text.ts:129**

`copy.ts` guards the DOM attribute with `copySeqMap` so a slow earlier write cannot take it back (README:603 guarantees exactly that). The composable's `copyState` ref is written from a bare `.then` with no sequence check, so the same race the attribute is immune to lands on the ref. Two mirrors of one state, one of them unguarded, and the guarded one is the one the tests assert.

*Symptom:* Verified: two `copy()` calls in flight, the second resolves and the first then rejects — `el.getAttribute('data-select-text-copy')` stays `'copied'` while `api.copyState.value` ends at `'error'`. A UI bound to copyState shows a red failure cross for a copy that succeeded, next to CSS driven off the attribute showing the green tick.

### 7. clear() is silently undone by a write already in flight  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/use-select-text.ts:155**

`clearApi` sets `copyState` to `'idle'` but does nothing about the outstanding `writeText` promise, whose `.then` unconditionally writes `'copied'`/`'error'` afterwards. The directive path has `abandonCopy`/`copyAbandoned` for precisely this; the composable has no equivalent. README:674 says copyState is "Reset by clear()" — it is reset, then un-reset a tick later.

*Symptom:* Verified: `copy()`, then `clear()` → copyState `'idle'`; the write lands → copyState `'copied'` while `state` is `'idle'`. A badge reading copyState flashes "Copied!" after the user pressed Clear.

### 8. buildTextMap crashes on a missing getComputedStyle that the diagnostics carefully guard  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/text-map.ts:153**

`warnIfUnselectable` and `warnIfNotRendered` both open with `typeof window.getComputedStyle !== 'function'` — so the package explicitly anticipates the call being absent. The load-bearing path that actually needs it, and runs first, has no such check, and `selectTextHost`'s try/catch only wraps `createRange`/`setStart`, not `buildTextMap`. Relatedly, `isInlineBox` does `display.startsWith('inline')` on whatever the style object hands back, so a stub without `display` is a TypeError too — which is why the `user-select: none` test block at vSelectText.text.test.ts:812 survives only because its fixtures contain no element children.

*Symptom:* Verified: with `getComputedStyle` unavailable, `mounted` throws `window.getComputedStyle is not a function` straight out of the lifecycle hook instead of no-opping the way the guarded warnings would. Any environment that stubs or narrows `getComputedStyle` (test harnesses, some embedded webviews, a consumer's own spy) turns a selection into a render error.

### 9. Both primary exports carry no JSDoc — it is attached to a private helper and to a params interface  ·  `readability`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/directive.ts:15**

The block documenting `v-select-text` (its examples, its `@see`) sits immediately above a *second* block documenting `syncCopy`, so the directive's docs belong to a private function and `export const vSelectText` at line 47 has nothing. Same mistake in use-select-text.ts:16: the "Imperative companion" block with the usage example lands on `export interface UseSelectTextParams`, and `export function useSelectText` at line 78 is undocumented. That example is also broken — it declares `const inputRef = ref(...)` and then passes `() => quoteRef.value`, a variable that does not exist. In a package whose stated distribution channel is people reading and copying these files, the two things a reader hovers first explain nothing.

*Symptom:* A consumer hovers `v-select-text` or `useSelectText` in their editor and gets a bare type signature; the copy-pasteable example they do find references an undeclared `quoteRef` and does not compile.

### 10. playground.html is a stale 1.x reimplementation of the directive, and a test calls it canonical  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/playground.html:66**

The file inlines its own `resolveBinding`, `performSelection`, `previousConditionMap` and `isSelectable` — a `condition`-only, inputs-and-textareas-only directive that warns "Directive only works on <input> and <textarea> elements", i.e. the exact 1.x behaviour README:733 says was removed. It does edge detection on the previous `condition` value, which is the bug SEL-4 fixed. Meanwhile playground.smoke.test.ts:3 opens with "The `playground.html` demo is the canonical 'does the library work in a real Vue app' exhibit" and then never loads it, building its own inline app instead.

*Symptom:* Someone opening the package to see how the directive works reads a second, contradictory implementation that is two major behaviours out of date, and a test file tells them it is the canonical one.

### 11. 18 tests × 2 Vue versions assert responsiveness against a value nothing reads  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/playground.smoke.test.ts:39**

`setViewport` only defines `window.innerWidth`/`innerHeight`. Nothing in `src/` reads either, and jsdom has no layout, so the mobile/tablet/desktop describes execute byte-identical code paths. The suite's stated purpose — "the three responsive breakpoints we claim to support" — cannot fail for a layout reason, because no layout is involved.

*Symptom:* A green "responsive smoke" suite is read as evidence the directive was exercised at three viewport sizes; a real breakpoint regression (the library has none today, but any future style-dependent path would) passes all 54 assertions.

### 12. Two copy-paste demos tell strangers the directive never focuses; the code says it does  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-select-text/06-input.vue:30**

06-input.vue says "The directive selects; it never focuses" at line 30 and repeats it at line 157. But `selectInputOrTextarea`'s whole-value path calls `el.select()`, and selection.ts:49 states in its own comment that "`el.select()` would still focus it" — the author's measured claim, used there as the reason to skip the empty case. Both cannot be true. jsdom's `select()` does not focus (I confirmed: activeElement is unchanged), so no unit test can arbitrate, and every demo handler calls `.focus()` itself first, so the browser checks cannot see it either. The `setSelectionRange` path genuinely does not focus, so the directive's focus behaviour silently depends on which branch ran.

*Symptom:* `<input v-select-text :value="x">` inside a modal or a long page steals focus on mount (and scrolls to itself), and so does `{ start, end }` on `type="number"`/`type="email"` via the `select()` fallback, while the identical binding on `type="text"` does not. The README documents neither, and the demo a developer copies states the opposite.

### 13. visibility:hidden text shifts every offset, and ARCHITECTURE says hidden subtrees are skipped  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/text-map.ts:117**

`walkRendered` skips only `display: none`. ARCHITECTURE.md:35 claims "hidden and non-rendered subtrees skipped" and README:284 claims "Subtrees that paint no text are skipped"; `visibility: hidden` paints no text and is counted as fully rendered. Since the package's entire selling point is that offsets are counted against the text *as rendered*, an unchecked case here moves every offset after it.

*Symptom:* Verified: `<span style="visibility:hidden">GHOST</span>visible` resolves to `"GHOSTvisible"`, so `{ start: 0, end: 5 }` selects the invisible word and every later offset is off by five. A screen-reader-only or animating-out sibling quietly breaks an offset range that was correct yesterday, with no warning.

### 14. The "warn once" sets memoize only the failure case; the happy path re-walks the DOM every cycle, in production  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/element-kind.ts:118**

`warnIfNotRendered` walks the entire ancestor chain calling `getComputedStyle` on each element, and only adds to `notRenderedWarned` when it *finds* a hidden ancestor — so for the normal visible host it is never memoized and runs in full on every selection cycle. `warnIfUnselectable` returns early before touching its WeakSet for the same reason. On top of that `buildTextMap` calls `getComputedStyle` once per element child. There is no `NODE_ENV`/`__DEV__` guard anywhere in `src/`, so all four diagnostics ship to production with no way to disable them. The names (`notRenderedWarned`, `unselectableWarned`) read as caches; they are not.

*Symptom:* A `trigger: 'always'` host ten levels deep forces twelve-plus style resolutions per render in the shipped bundle, on a path whose only product is a console warning that will never be printed. Profiling blames "style recalculation" and nobody connects it to a selection directive.

### 15. A range request on an input that rejects setSelectionRange is silently upgraded to "the whole value"  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-select-text/src/selection.ts:69**

When `setSelectionRange` throws (real behaviour on `type="number"`/`type="email"`), the catch calls `selectAll()` — reported honestly on the event, but it means "select these five characters" becomes "select all thirty", chosen by which branch threw rather than by anything the consumer asked for. With `copy: true` the clipboard receives the whole field. README:728 documents the fallback; neither the README nor the types mention what it does to `match` or to `copy`.

*Symptom:* `<input type="email" v-select-text="{ match: localPart, copy: true }">` puts the user's full email address on the clipboard instead of the local part, reporting `start: 0, end: value.length` — technically truthful, and nobody reads the detail.

## Comments and docs the code contradicts

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-select-text/06-input.vue:30 — "The directive selects; it never focuses" — the select() path focuses, per the package's own comment at src/selection.ts:49

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-select-text/06-input.vue:157 — same claim repeated in the card's prose, on a card whose fourth block exercises the select() fallback

- /Users/ozgurseyidoglu/Development/npm/v-select-text/README.md:284 — "A match can never reach into text the user cannot see" — false under whitespace:'preserve', and contradicted by README.md:294 ten lines later

- /Users/ozgurseyidoglu/Development/npm/v-select-text/ARCHITECTURE.md:35 — "hidden and non-rendered subtrees skipped" — only display:none is skipped; visibility:hidden is counted, and 'preserve' skips nothing at all

- /Users/ozgurseyidoglu/Development/npm/v-select-text/ARCHITECTURE.md:89 — "Copy-paste consumers: every file under src/ plus the entry ... take the folder as-is" — package.json files:["dist"] means src/ is never in the tarball

- /Users/ozgurseyidoglu/Development/npm/v-select-text/src/types.ts:89-92 — "`>` the text length is clamped to the text length" — Infinity is not clamped, it becomes "select everything"

- /Users/ozgurseyidoglu/Development/npm/v-select-text/src/resolve.ts:2 — module header says "numeric offsets are clamped into range"; resolveBinding clamps nothing (start/end pass through raw), find-range.ts does the clamping

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-select-text/manifest.ts:16 — "~4,700 real writes per second" for the SEL-5 loop, where copy.ts:72, state.ts:76, types.ts, README:534, the tests and demo 16 all say "~8,600" (and demo 16 also says "41,461 in five seconds"). Two different measured numbers for one measurement.

- /Users/ozgurseyidoglu/Development/npm/v-select-text/vSelectText.test.ts:786 — describe("binding.oldValue prev-tracking (no module state, post-refactor)") — the directive reads module state (edgeSpentMap) and never reads binding.oldValue

- /Users/ozgurseyidoglu/Development/npm/v-select-text/vSelectText.test.ts:806 — "update with binding.oldValue=undefined treats prev as derivable (handles HMR initial update)" — oldValue is not read by any code path

- /Users/ozgurseyidoglu/Development/npm/v-select-text/vSelectText.text.test.ts:1006 — "a detached host reports nothing, because addRange silently aborts" — selection.ts:116 returns at !el.isConnected, long before addRange

- /Users/ozgurseyidoglu/Development/npm/v-select-text/playground.smoke.test.ts:1-21 — "Playground responsive smoke test ... the three responsive breakpoints we claim to support" and "playground.html ... is the canonical exhibit": no breakpoint is exercised and playground.html is never loaded

- /Users/ozgurseyidoglu/Development/npm/v-select-text/vitest.workspace.ts:6 — names three test files for the vue-3.5/vue-3.3 projects; the include lists hold five

- /Users/ozgurseyidoglu/Development/npm/v-select-text/src/copy.ts:51 and src/click-trigger.ts:8 — shipped source cites "SEL-2", "SEL-4", "SEL-5" and "DESIGNS → DZ-1 / A11Y-1"; those documents live at the repo root and are in neither the package nor the tarball, so for any reader of src/ they resolve to nothing

- /Users/ozgurseyidoglu/Development/npm/v-select-text/src/use-select-text.ts:40 — "Initial / current options" — params.options is copied once at construction and never re-read; update() is the only door

## ARCHITECTURE.md claims nothing enforces

- ARCHITECTURE.md:26 — 'selection.ts is the only place a selection happens, and selectAndReport is its only export that selects.' use-select-text.ts:143 calls el.setSelectionRange(0, 0) and :150 calls sel.removeAllRanges() outside that file, mutating the selection with no event. Nothing prevents it; the module-private split only protects the install half.

- ARCHITECTURE.md:70-77 — 'applyRange opens with removeAllRanges(), so a host that resolved to nothing would otherwise wipe whatever the user had selected. The ordering is the invariant.' useSelectText().clear() calls removeAllRanges() unconditionally, with no check that this host owns the current selection — verified to destroy an unrelated selection. The invariant is enforced in one file and violated in its sibling.

- ARCHITECTURE.md:34-38 — 'text-map.ts is the only place that walks text nodes. Offsets are expressed against the text as rendered — hidden and non-rendered subtrees skipped.' True only for display:none, and only in collapse mode. visibility:hidden is counted; whitespace:'preserve' skips nothing at all, including <script>, <style> and display:none. Nothing in the module distinguishes the two modes' guarantees, and no test asserts the invariant as written.

- ARCHITECTURE.md:88-89 — 'Copy-paste consumers: every file under src/ plus the entry is self-contained TypeScript ... take the folder as-is.' package.json files:["dist"] means src/ never ships, and the shipped source (when read in the repo) cites SEL-2/SEL-4/SEL-5 and DESIGNS → DZ-1 / A11Y-1, none of which travel with the folder. Nothing checks either half.

- ARCHITECTURE.md:42-51 — 'copy.ts is the only place a clipboard write happens, and the write is always initiated in the same synchronous turn as the selection it copies.' Holds today, but the only thing protecting it is that startCopy is called from exactly one place; it is exported (vSelectText.copy.test.ts:19 imports it directly) with no guard that the caller is in the selection turn.

- README.md:598 — 'One selection = at most one attempt = at most one select-text-copy.' Holds for the directive; the composable's copy() can have several attempts in flight whose settle order is unguarded for copyState (verified), so the derived reactive state does not honour the guarantee the attribute does.

- types.ts:8 and README:698 — the four-kind classification is claimed exhaustive, but getElementKind checks contenteditable (element-kind.ts:70) before the NON_TEXT_TAGS list (:71), so <img contenteditable> classifies as 'contenteditable' and takes the Range path instead of warning.

## Tests passing for the wrong reason

- playground.smoke.test.ts (all 18 tests, run twice) — the mobile/tablet/desktop describes only set window.innerWidth/innerHeight, which no file under src/ reads, and jsdom has no layout. Three identical code paths asserted as three breakpoints.

- vSelectText.test.ts:786-812 'binding.oldValue prev-tracking (no module state, post-refactor)' — passes because edgeSpentMap happens to agree with the oldValue the test passes. Every makeBinding(value, oldValue) in the file feeds a parameter the directive never reads, so a reader concludes the edge detector is oldValue-driven when it is WeakMap-driven.

- vSelectText.empty.test.ts:107, :120, :129, :136 and :276 — 'selects the text when it arrives' / 'stays armed across any number of empty updates' all prove the flagship promise by calling vSelectText.updated!() by hand. In a real app that hook is only guaranteed when the text is interpolated in the owning component; with the text owned by a child, zero events fire (verified). The tests manufacture the trigger the production code cannot guarantee.

- vSelectText.text.test.ts:812-868 ('user-select: none' diagnostic) — stubs window.getComputedStyle globally with { userSelect } only. It survives solely because the fixtures are text-only hosts: one element child would reach isInlineBox(undefined).startsWith and throw. The stub also silently disables the whiteSpace read in buildTextMap for the duration.

- vSelectText.copy.test.ts:448-473 ('copy.ts: the module boundary refuses an empty string on its own') — reaches the 'empty' branch by importing startCopy directly and hand-building a detail; the comment itself states no public binding can get there. README:521 lists 'empty' as a public reason a consumer may observe, which nothing can produce.

- vSelectText.text.test.ts:1005-1011 — asserts a detached host fires nothing and credits addRange aborting silently; the code returns at the isConnected check and the addRange path is never entered, so the test cannot regress if that guard is deleted (the isConnected check would still catch it — but if *it* were deleted, the test's stated mechanism is untested).

- vSelectText.copy.test.ts:914 ('one unchanged text = one write, however many times the handler re-renders') — asserts rig.renders() > 0 as proof 'the loop's driver is genuinely live'. With the guard in place the handler fires exactly once, so exactly one re-render occurs; the assertion is satisfied by a single render, not by a live loop.
