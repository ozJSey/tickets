# Quality audit — `v-keyboard-navigation`

**Verdict: needs-work**  ·  19 findings

**PUBLISHED ON NPM — defects here are live for real consumers.**


## The single worst thing

The controlled scroll — the single thing the README, the CHANGELOG and scroll.ts's own header all name as the reason this package exists — is aimed at the wrong element. `resolveContainer` turns a bare CSS selector into `document.querySelector(ref)`: the FIRST match in the whole document, not the one belonging to this group. Two instances of the same component on one page, and arrowing inside the second one scrolls the first one's pane — I measured it (`scrollTo({top:340})` landed on instance one; instance two never moved). The `:scope` form that looks like the scoped escape hatch is no better: it is `el.closest(sel)`, which walks ANCESTORS, crosses the host boundary, and means the opposite of what `:scope <sel>` means in CSS. Every existing test mounts exactly one host, so neither can ever be caught. This is the exemplar shape: the decision — which box scrolls, and what `relTop` is relative to — is fed a stranger, and the symptom ("the list doesn't follow the focus" / "an unrelated panel jumped") points nowhere near scroll.ts.


## Findings

### 1. A bare `container` selector resolves document-globally, so the wrong element scrolls  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/scroll.ts:41**

`document.querySelector(ref)` returns the first match in the document. Any component rendered more than once — a listbox in a repeated card, two panes side by side — has every instance resolve to instance one's container. The maths downstream (`relTop = itemRect.top - boxRect.top + container.scrollTop`) is then relative to a box that does not contain the item, so it writes an arbitrary scrollTop onto a foreign element. Scoping it to the host is one call (`host.querySelector`), and the README's own wording ("Pin the scroller") implies it is already scoped.

*Symptom:* Two instances of the same listbox component on a page: arrowing in the second silently scrolls the first (measured: `{top:340}` written to pane one, pane two untouched), and the second list never follows its own focus ring.

### 2. `:scope <sel>` is implemented as `closest()`, which inverts CSS semantics and escapes the host  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/scroll.ts:40**

In CSS `:scope .pane` means "a DESCENDANT .pane". Here it means "the nearest ANCESTOR .pane", found with `closest`, which walks past the host and out into the rest of the document. So the one form documented as the scoped alternative to a global selector is neither scoped nor the direction its syntax says. A stranger copying the README line `container: ':scope .scroll-pane'` will point it at a descendant pane and get `null` — a documented silent no-op.

*Symptom:* Verified: a `.pane` two levels ABOVE the directive host was resolved and scrolled, while a `.pane` inside the host would silently resolve to null and scroll nothing at all.

### 3. `enabled: false` does not disable the focus listeners, so the state attribute stops saying `disabled`  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/directive.ts:85**

`onKeydown` checks `group.opts.enabled`; `onFocusin` and `onFocusout` do not. `release()` restored every original tabindex, so the controls are natively focusable again — one click inside a disabled group fires focusin, which sets `hasFocus` and writes state `active`. Clicking away writes `group.items.length === 0 ? 'empty' : 'idle'`, and after release `items` is empty, so the host now reports `empty` — the one value the package documents as "no focusable items, the group has left the keyboard". It only heals on the next subtree mutation. types.ts:126 says `false` leaves "listeners idle", which is false for two of three.

*Symptom:* Confirmed in the probe: disabled → click a cell → state `active` → click away → state `empty`, permanently. In playground demo 11-api.vue, `.strip[data-keyboard-navigation-state='disabled'] { opacity: 0.5 }` un-greys the moment you click it, and the card's own prose ("sets data-keyboard-navigation-state='disabled' — the state is reported rather than silently assumed") is falsified by a single click. CSS keyed on `empty` as an alarm fires on a group that is merely switched off.

### 4. Focus rescue treats any null-relatedTarget blur as a removal, so it steals focus back from the browser  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/directive.ts:100**

`strandedItem` is set whenever `relatedTarget === null`. That is Chrome's signal for "the focused element was removed", but it is also what you get for a window blur, a Tab into the browser chrome, and (in Safari/Firefox) a click on non-focusable page chrome. The next sync that drops the previously active item treats it as a stranding and calls `activate(...)`, moving DOM focus. The package cannot distinguish "focus went nowhere because I removed the element" from "focus went nowhere because the user left". The existing test only covers the case where relatedTarget is a real element.

*Symptom:* User tabs to the URL bar or switches windows; an async list refresh a moment later drops the row they were on; focus is yanked back into the widget. Verified: activeElement went from `<body>` to the rescued row with no removal-driven blur in play.

### 5. A nested group's host is silently dropped from its parent's items, creating a second tab stop  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/items.ts:77**

`el.closest(HOST_SELECTOR) !== host` skips any descendant whose nearest group host is not this host — but an element that is ITSELF a group host is its own nearest host, so it is skipped by its parent too. The rule needed is "skip if a host sits strictly between el and host". The dropped element keeps its natural tabbability, so the parent group now has its own roving `0` plus a second native tab stop in the middle of it: exactly the two-tab-stop failure ARCHITECTURE.md invariant 1 and README's "Exactly one tabbable item, always" section exist to prevent, caused by the package's own nesting rule.

*Symptom:* Confirmed: a menubar whose middle menuitem also carries the directive collected items `['File','View']` — 'Edit' vanished from arrow navigation and became a second Tab stop. No error, no warning, and the flagship invariant is broken by the library itself.

### 6. scroll.ts writes `style`, which roving.ts observes — the module that must not be self-referential is  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/scroll.ts:138**

`'style'` is in `OBSERVED_ATTRIBUTES` (roving.ts:41). When `offset` is set, every keystroke writes `style.scrollMarginTop` and then restores it — two unguarded style mutations on an item inside the observed subtree — so the group's own MutationObserver wakes and runs a full `sync()` (re-resolve, re-collect, re-apply) after every arrow key. ARCHITECTURE.md invariant 3 states exactly this hazard ("an unconditional setAttribute produces a mutation record even when nothing changed — which is an observer loop") and assigns it to state.ts, but scroll.ts does it anyway. The `finally` restore also assumes `scrollIntoView({behavior:'smooth'})` snapshots scroll-margin synchronously — true in Chromium, unspecified.

*Symptom:* Measured: 3 keystrokes with `offset` set produced 3 wakeups of an observer with the same attributeFilter; without `offset`, 0. So held-arrow key repeat runs a full re-sync per frame, and the suite's 'does not loop' test uses no offset and therefore cannot see it.

### 7. `offset` leaves `style=""` permanently on every item it touched, including after unmount  ·  `leak`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/scroll.ts:146**

Clearing `style.scrollMarginTop` leaves the `style` attribute present-but-empty. `release()` is documented as "Give the DOM back: original tabindex, no generated ids, no item hooks" and restores three things — not this. The package has a test named 'leaves the DOM as it found it on unmount'; it does not use `offset`.

*Symptom:* Confirmed: items keep `style=""` after unmount. Any CSS rule, snapshot test or `[style]` selector in the host app sees a changed DOM that nothing in the package admits to writing.

### 8. PageUp/PageDown are measured on the block axis regardless of the axis being navigated  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/paging.ts:31**

`pageStep` only ever reads `viewport.clientHeight` and item `height`. `group.opts.axis` is never passed in. On a horizontally scrolling toolbar/tablist every item's height equals the strip height, so `used + box > height` trips on the second item and the step is 1. The module's own doc promises "count how many items fit the viewport" and "degrades to first/last when nothing scrolls" — on the inline axis it does neither. `scrollParent` compounds it by regex-testing `overflowY + overflowX` concatenated, so an element that scrolls only horizontally is accepted as the vertical page viewport.

*Symptom:* Measured: PageDown in a 300px-wide strip of 40px items moved from A to B — one item, indistinguishable from ArrowRight. The README sells this key as "a real visible page ... not mapped to first/last the way Radix and Reka do"; on a horizontal group it is worse than either.

### 9. Text inputs are default items but keys.ts refuses to navigate off them — an arrow-key dead end  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/items.ts:33**

`FOCUSABLE_SELECTOR` contains bare `'input'`, so `<input type="text">` becomes an item and receives the roving `0`. `isTextEntry` then returns true for it, so `intentFor` answers `null` for every arrow — correctly, the input owns its keys. The two rules are each locally right and jointly produce a trap: the group is one tab stop, so Tab leaves it entirely, and the arrows will not move off the input. Nothing in the README or the options table warns about this; the pattern table even says Combobox is out of scope "because it owns an input's keys, which this must never do" — while the default selector swallows inputs anyway.

*Symptom:* Confirmed: ArrowRight from a toolbar button onto a filter input, then ArrowRight again is not claimed and focus stays put (input keeps `tabindex="0"`). The user cannot reach the rest of the toolbar without Shift+Tab and re-entering. The package's own test at vKeyboardNavigation.test.ts:821 asserts this state and calls it key discipline.

### 10. `onFocusout` ignores `ownsEvent`, the rule the helper above it states  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/directive.ts:93**

`ownsEvent` is documented "Events from a nested group belong to that group, not to this one" and is called by `onKeydown` and `onFocusin`. `onFocusout` only checks `relatedTarget` containment. So a blur out of a NESTED group runs the outer group's focusout path: it clears the outer typeahead buffer, writes the outer host state, sets the outer `strandedItem` to an element that is not even in its item set, and — with `memory: false` — resets the outer tab stop and emits an `onNavigate` with reason `sync` that the consumer did not cause.

*Symptom:* A menubar containing a menu: tabbing out of the submenu fires a spurious `keyboard-navigate` on the menubar and moves the menubar's tab stop back to its initial item, so Shift+Tab returns the user to the wrong control.

### 11. Every focusin is reported as `reason: 'pointer'`, including Tab and programmatic focus  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/directive.ts:90**

`onFocusin` hard-codes `'pointer'`. focusin fires for keyboard Tab, for `el.focus()` from application code, and for a programmatic restore — none of which is a pointer. The README publishes the union `'key' | 'typeahead' | 'pointer' | 'api' | 'sync'` with no note, and playground 12-events-and-state.vue tells the reader "The reason tells them apart". The package's own test at :1096 uses a bare `.focus()` call to assert `'pointer'`, enshrining the mislabel.

*Symptom:* Confirmed: a synthetic focusin with no pointer involvement reported `'pointer'`. A consumer writing `if (reason === 'pointer') track('click')` logs a click for every Tab into the widget.

### 12. The README's headline scroll trace and scroll.ts's contradict each other  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/scroll.ts:11**

scroll.ts's module header (the one a copy-paste reader reads) gives `0,0,0,0,0,120,120,120,240,240,240,360` and `0,0,0,0,0,40,80,120,160,200,240,280`. README.md:41-43 and CHANGELOG.md:48-51 give `0,0,0,0,120,...,360,360` and `0,0,0,0,40,...,280,320` — shifted by one step. Both claim to be the same measurement at the same geometry in the same Chrome build. One is wrong and nothing says which. CHANGELOG.md:52 then claims the interaction spec "re-measures the first and third of those ... so the claim is re-checked on every run rather than quoted"; the spec actually asserts `out.max === 40` and `out.max >= 100` — the shape, not the traces — so neither table is re-checked, and they could not both be.

*Symptom:* The package's central evidence cannot be reproduced from its own documentation, and a reader who notices the discrepancy has no way to tell which number set to trust.

### 13. ARCHITECTURE.md and README both claim scroll/typeahead/paging import nothing from siblings; scroll.ts does  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/ARCHITECTURE.md:57**

"`scroll.ts`, `typeahead.ts` and `paging.ts` are useful on their own and import nothing from their siblings" — repeated verbatim at README.md:299. scroll.ts:29 imports `KeyboardNavigationContainer` and `ResolvedScroll` from `./types`. Type-only, so it compiles away, but the claim is aimed precisely at the reader who lifts one file, and that reader gets an unresolved import. Separately, the build entry's own header (vKeyboardNavigation.ts:5) points at "`src/ARCHITECTURE.md` at the repo root" — the file is `ARCHITECTURE.md` at the package root, so the first pointer a copy-paste reader follows is wrong in both the directory and the level.

*Symptom:* Someone lifts scroll.ts into their project on the strength of the claim and gets a TypeScript error on line 29, then goes looking for `src/ARCHITECTURE.md` at the repo root and finds nothing.

### 14. items.ts's skipping rules list omits two of the rules it implements  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/items.ts:8**

The module doc enumerates the skip conditions as "`[disabled]`, `[aria-disabled="true"]`, `[hidden]`, `[inert]` and an inline `display: none` / `visibility: hidden`". The code also skips on `aria-hidden="true"` anywhere up the chain to the host (items.ts:61) and on `closest('fieldset[disabled]')`, which walks ABOVE the host and out of the group (items.ts:52). Neither appears in the doc, the README's accessibility notes, or the CHANGELOG's limitations. The package's own test at :545 asserts the aria-hidden behaviour, so it is deliberate and simply undocumented.

*Symptom:* A group wrapped in a decorative `aria-hidden="true"` container, or any group inside a disabled `<fieldset>`, reports `data-keyboard-navigation-state="empty"` with zero explanation — the documented alarm state firing for a reason the documentation does not list.

### 15. Observing the `checked` attribute cannot see a radio being checked  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/roving.ts:47**

`'checked'` is in `OBSERVED_ATTRIBUTES` alongside `aria-checked`, and `initialIndex` reads `el.matches(':checked')`. But user interaction and Vue's `v-model` change the `checked` IDL property, not the content attribute, so no mutation record is ever produced. The observer entry only fires for an explicit `:checked` attribute binding. The group's idea of where the selection is goes stale and is only repaired incidentally, by the focusin that accompanies the click.

*Symptom:* `memory: false` on a native radio group resets the tab stop to `initialIndex`, which reads a `:checked` state the observer never learned about — the tab stop lands on a radio that is not the checked one.

### 16. Demo 07 hand-copies ROLE_DEFAULTS and then forces a remount, so it cannot show what it claims  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-keyboard-navigation/07-wrap-and-orientation.vue:10**

`DEFAULTS` at line 10 is a second, hand-maintained copy of `ROLE_DEFAULTS` / `NO_ROLE_DEFAULTS` from roles.ts — the exact two-writers pattern this package's own ARCHITECTURE preaches against, in a file written to be copied. Change a role default in the library and the card silently keeps displaying the old one. Worse, line 52 puts `:key="role"` on the host, which tears down and remounts the directive on every role change — so the card whose job is to demonstrate "changing `role` at runtime takes effect without remounting" (resolve.ts:4-5) never exercises the `updated` → re-resolve path at all.

*Symptom:* The only live demonstration of runtime re-resolution is a remount in disguise; if re-resolution regressed, every test and every card would still be green.

### 17. Demo 06 ships the 2D grid the README calls "the worst outcome"  ·  `readability`

**/Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-keyboard-navigation/06-typeahead.vue:31**

`role="listbox" aria-orientation="horizontal"` on a `grid-template-columns: repeat(4, 1fr)` of 12 cells. `aria-orientation` beats the listbox block default, so the axis is `inline`: Up and Down do nothing at all, and ArrowRight at the end of a visual row jumps to the start of the next one — visually down-and-left. README.md:91 says "Grid / 2D ❌ not implemented — deferred, and a half-done grid is the worst outcome"; this card is a half-done grid offered as a copy-paste example.

*Symptom:* Someone copies the typeahead card, gets a 2D grid where vertical arrows are dead keys, and concludes the library is broken rather than that the card chose the wrong axis.

### 18. Demo 08's re-enable branch is unreachable from its own UI  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-keyboard-navigation/08-dynamic-list.vue:47**

`toggleDisabledFocused` reads `document.activeElement.textContent` and toggles the row in/out of the `disabled` list. But disabling a row makes the directive release its tabindex, so the row stops being focusable and can never be the active element again — the `disabled.value.filter(...)` branch can never run. The row's text also gains " (disabled)" (line 79), so even a contrived path would fail the `rows.value.includes(row)` guard at line 46. The button is labelled "Disable focused" but reads as a toggle in the code.

*Symptom:* A reader copies the toggle, ships a "disable/enable row" control, and the enable half never fires. In the card itself, Refill is the only way back.

### 19. The browser interaction spec — the package's stated proof layer — runs on 15 fixed sleeps  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/playground/scripts/interactions/v-keyboard-navigation.mjs:1**

`__kn.sleep(200)` / `sleep(250)` / `sleep(300)` gate 15 of the 21 checks, standing in for "Vue rendered, then the MutationObserver microtask ran, then the scroll settled". There is no condition polled anywhere. CHANGELOG.md:52 and the manifest lean on this spec as the browser-side evidence for the package's headline claim, so the evidence is timing-dependent rather than sequenced — on a loaded CI box a slow render reads the previous state and the check passes or fails for reasons unrelated to the code.

*Symptom:* The scroll-trace check reads `scrollTop` before the last keystroke has settled and passes with a trace one key short, or fails on a slow machine and gets retried until green — either way the claim is no longer verified.

## Comments and docs the code contradicts

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/scroll.ts:11 — the measured scrollTop trace here (5 leading zeros, tail …240,360) contradicts README.md:41-43 and CHANGELOG.md:48-51 (4 leading zeros, tail …360,360). Same geometry, same Chrome build, different numbers.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/types.ts:126 — "`false` releases the group: tabindex restored, listeners idle." The keydown listener idles; focusin and focusout stay fully live and rewrite the host state attribute.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/directive.ts:79 — "Events from a nested group belong to that group, not to this one." `onFocusout` (directive.ts:93) never calls `ownsEvent`, so nested-group blurs do run on the parent.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/items.ts:8 — the enumerated skip rules omit `aria-hidden="true"` (items.ts:61) and `closest('fieldset[disabled]')` (items.ts:52), both of which the code applies and the README also never mentions.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/paging.ts:14 — "When nothing scrolls ... the step degrades to first / last." On the inline axis it degrades to a single item (measured: A→B in a 300px strip of 40px items), not to first/last.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/paging.ts:6 — "the viewport is injected so the caller owns the 'which element scrolls' question". The caller owns which element; nobody owns which AXIS — `clientHeight` and item `height` are hard-coded and `opts.axis` is never passed in.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/scroll.ts:53 — "Only used to size a PageUp / PageDown step — the scroll itself never needs it." True, but the same function's regex tests `overflowY + overflowX` concatenated, so it returns a horizontal-only scroller as the vertical page viewport.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/ARCHITECTURE.md:57 and README.md:299 — "`scroll.ts`, `typeahead.ts` and `paging.ts` ... import nothing from their siblings." scroll.ts:29 imports from `./types`.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/ARCHITECTURE.md:42 — invariant 3, "`state.ts` owns every DOM reflection, and every write is guarded". scroll.ts:138 writes `style` unguarded (twice per keystroke, and `style` is an observed attribute); roving.ts:278 writes `item.id` directly.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/vKeyboardNavigation.ts:5 — points the copy-paste reader at "`src/ARCHITECTURE.md` at the repo root"; the file is `ARCHITECTURE.md` at the package root.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/keys.ts:70 — "Somebody upstream already dealt with it — including another instance of this directive in a nested group." The nested case is already handled by `ownsEvent` before `intentFor` is reached; the stated reason never applies.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/src/roving.ts:194 — "Give the DOM back: original tabindex, no generated ids, no item hooks." It does not give back the `style=""` attribute scroll.ts leaves on every item when `offset` is set.

- /Users/ozgurseyidoglu/Development/npm/v-keyboard-navigation/CHANGELOG.md:52 — "The playground's interaction spec re-measures the first and third of those against the live card ... so the claim is re-checked on every run rather than quoted." The spec asserts `out.max === 40` / `out.max >= 100`; it never compares against either quoted trace.

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-keyboard-navigation/11-api.vue:45 — "sets `data-keyboard-navigation-state="disabled"` — the state is reported rather than silently assumed." Clicking a cell in the disabled strip flips it to `active`, then `empty`.

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-keyboard-navigation/12-events-and-state.vue:54 — "The `reason` tells them apart: key, typeahead, pointer, api, sync." `pointer` is emitted for keyboard Tab and for programmatic focus as well.

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-keyboard-navigation/05-radiogroup.vue:40 — "Home, End and typeahead have no native behaviour there, so those still work." They move focus among native radios without moving the check, which is the exact harm the card gives as the reason arrows are left alone.

## ARCHITECTURE.md claims nothing enforces

- Invariant 1, "`roving.ts` is the only writer of `tabindex`, and the only place focus moves ... Exactly one item is tabbable at every instant." Nothing prevents a group host from being skipped as its parent's item (items.ts:77), and that dropped element keeps its natural tabbability — a menubar with a nested group has two tab stops. Verified. The invariant is also only ever checked inside one group: no code, and no test, counts tab stops across a nesting boundary.

- Invariant 2, "`scroll.ts` is the only place a scroll happens, and it always runs *after* the focus call ... That ordering is enforced in `roving.ts:activate`, which is the only caller." The ordering is real, but nothing enforces singleness of the caller — it is a convention, and a second call site would compile. More to the point, the invariant says nothing about WHICH element is scrolled, which is where the package actually breaks (scroll.ts:41).

- Invariant 3, "`state.ts` owns every DOM reflection, and every write is guarded (`writeAttr` skips a write that would not change the value)." Two writers exist outside state.ts and neither is guarded: scroll.ts:138/139 writes `style` (an OBSERVED attribute, so this is precisely the observer-loop hazard the invariant was written to prevent), and roving.ts:278 writes `item.id` via the property. Nothing in the build, the types or the tests prevents a third.

- Invariant 4, "`keys.ts` is the only place that decides a keystroke is ours. It answers `null` for everything else." Mostly held, but `directive.ts:118` short-circuits Escape before `intentFor` is ever consulted, so there is a second decision site — benign today (it does not preventDefault) and unmentioned by the invariant.

- ARCHITECTURE's "For the copy-paste reader: every file under `src/` ... Take the folder as-is, or lift one module: `scroll.ts`, `typeahead.ts` and `paging.ts` ... import nothing from their siblings." scroll.ts imports `./types`. There is no lint rule, no dependency-cruiser config and no test asserting the import graph, so the module map in ARCHITECTURE.md is a description of intent that the build would not notice drifting.

## Tests passing for the wrong reason

- vKeyboardNavigation.test.ts:637 'scrolls a pinned container itself instead of the ancestor chain' — uses `container: '.host'` in a document containing exactly one `.host`, so `document.querySelector` happens to find the right element. No test in the suite can ever mount two instances, which is why the document-global resolution (the worst defect in the package) is invisible to 99 green tests.

- vKeyboardNavigation.test.ts:516 'does not loop: its own writes produce no further observer work' — runs without `offset`, so scroll.ts never writes `style` and the one observed attribute the package writes itself is never exercised. Adding `offset` makes the group's own observer fire once per keystroke (measured: 3 wakeups for 3 keys).

- vKeyboardNavigation.test.ts:821 'never intercepts inside a text input' — asserts `document.activeElement` is still the input after ArrowDown/Home/'b'. It is reading a keyboard dead end (the input is a roving item you cannot arrow off) and recording it as key discipline.

- vKeyboardNavigation.test.ts:1096 'names typeahead and pointer moves as such' — establishes `reason === 'pointer'` by calling `el.focus()` programmatically. No pointer is involved, so the test locks in the mislabel rather than verifying it.

- vKeyboardNavigation.test.ts:1209 'enabled: false gives the DOM back and says so' — presses a key but never focuses into the disabled group, so it misses that one focusin rewrites the state to `active` and the following focusout to `empty`. The 21-check browser spec (interactions/v-keyboard-navigation.mjs:518) has the same blind spot: it toggles the checkbox and reads the attribute without ever clicking inside.

- vKeyboardNavigation.test.ts:1315 'degrades to first / last where nothing scrolls' and the whole `pageStep` block — every case is vertical. Paging has no axis input at all, so the inline-axis behaviour (step of 1) is untested by construction, and the browser check at interactions:448 also only drives the vertical card.

- vKeyboardNavigation.test.ts:1263 'leaves the DOM as it found it on unmount' — no `offset` configured, so it never sees the `style=""` residue that survives unmount.

- vKeyboardNavigation.test.ts:920 'gives a nested group its own keys' — the nested host is a `<div>` that is not itself an item candidate, so the test never hits the case where a group host is also its parent's item and gets silently dropped (producing two tab stops).

- The two jsdom guards in scroll.ts:120 and scroll.ts:142 (`typeof container.scrollTo !== 'function'`, `typeof item.scrollIntoView === 'function'`) are unreachable in the suite: the harness installs both as spies in `beforeEach`. They are defensive branches for states the tests exclude, and CONVENTIONS bans exactly that.
