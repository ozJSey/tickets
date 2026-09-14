# Quality audit — `v-copy`

**Verdict: structurally-unsound**  ·  16 findings

**PUBLISHED ON NPM — defects here are live for real consumers.**


## The single worst thing

The binding object is simultaneously treated as consumer-owned read-only config AND as library-owned mutable state, and nothing in the code distinguishes the two. `resolve.ts:139` does `controllerObj = v as MutableController` for *any* plain object, so `v-copy="{ source: x }"` and `v-copy="ctrl"` are the same code path. From that one line flow four separate defects I reproduced: (a) the directive writes `history`/`copied`/`last`/`copy`/`clear` into the consumer's object, so a frozen config throws `TypeError: Cannot add property history, object is not extensible` and a `readonly()` config throws `TypeError: Cannot read properties of undefined (reading '0')` — both out of `mounted`, killing the component; (b) the config half is only ever read at render time, so `ctrl.disabled = true` does not disable anything until something unrelated re-renders the host; (c) `ctrl.last` is written by whichever object performed the copy rather than derived from the sink, so it goes stale in the README's own flagship picker pattern; (d) `ctrl.sink` is latched on first render and silently ignored forever after. The headline feature of the package — "slot-like reactive state with no composable" — is wired one-way, snapshot-based, and write-through-your-config. This is not a set of edge cases, it is the shape of the API.


## Findings

### 1. Any plain-object binding is adopted as a mutable controller — a frozen or readonly config crashes the mount  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/resolve.ts:139**

`controllerObj = v as MutableController` makes every object binding a controller, and `controller.ts:37` + `controller.ts:55-71` then WRITE `history`, `copied`, `last`, `copy`, `clear` onto it, unconditionally and without checking that the object is writable. `ensureControllerHistory` declares `: CopyEntry[]` but returns `ctrl.history` straight after the assignment — when the assignment is refused the return value is `undefined`, and `enrichController`'s `ctrl.last = history[0]` dereferences it. Reproduced: `v-copy="Object.freeze(cfg)"` → `TypeError: Cannot add property history, object is not extensible`; `v-copy="readonly(reactive(cfg))"` → `TypeError: Cannot read properties of undefined (reading '0')`. Both escape `mounted`. Freezing a module-level config object, or passing one down as a readonly prop / injected value, is ordinary Vue practice.

*Symptom:* A component blows up at mount with a TypeError about a property named `history` that the consumer never wrote, on a page whose only copy binding is `v-copy="COPY_CFG"`. Nothing in the message points at v-copy's ownership of the object.

### 2. Controller config is a render snapshot, not reactive — `ctrl.disabled = true` keeps copying  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/directive.ts:54**

`Resolved` is produced only in `mounted`/`updated`. The render function reads the controller *variable*, never its properties, so mutating `ctrl.disabled`/`trigger`/`dedupe`/`max`/`feedback` tracks nothing and triggers no re-render, so `updated` never fires and the old resolution stays live. Reproduced: with the README's controller example shape, after `ctrl.disabled = true` + `nextTick()` the element still copied; it only stopped once an unrelated `{{ tick }}` bump forced a patch. Same for `ctrl.trigger = false`. The playground knows: 13-history-picker.vue:41-47 and 15-dedupe-scope.vue:18-22 both carry paragraph-long comments explaining that the template must *read* the option or it "would flip silently and never take effect" — the defect is documented as a demo technique and appears nowhere in the README.

*Symptom:* "I set `ctrl.disabled = true` when the token was revoked and the element still copies it." Intermittent, because whether it takes effect depends on whether some other binding in the same component happened to change in the same tick.

### 3. `ctrl.last` is written by whichever binding object ran the copy, not derived from the sink  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/history.ts:73**

`record()` ends with `if (r.controllerObj) r.controllerObj.last = sink[0]` — it updates the `last` of the object that *performed* this copy. Any second binding that shares the same sink (exactly the README's picker: `v-copy="{ source: entry, sink: clipboard.history }"`, and demo 13's `sinkBinding()`) is a different object, so the real controller's `last` is never updated. Reproduced: after a row copy, `ctrl.history[0] === 'picked-from-row'` while `ctrl.last === 'first'`. `types.ts:152` documents `last` as "The most recent entry" and README.md:465 recommends it as the replacement for a primitive ref. This is the exemplar class — `last` depends on who called, when it should depend on the head of the array it claims to mirror.

*Symptom:* A "Last copied: {{ clipboard.last }}" line that silently freezes at the first value the moment the user picks anything from the history dropdown, while the list above it shows the correct order.

### 4. `ctrl.sink` is latched on the first render and ignored forever after  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/controller.ts:36**

`if (!Array.isArray(ctrl.history)) ctrl.history = Array.isArray(ctrl.sink) ? ctrl.sink : []` — once `history` exists, `sink` is never consulted again, and `directive.ts:35` then overwrites the freshly resolved `r.sink` with that cached array. Reproduced: after `ctrl.sink = secondArray` plus a forced re-render, copies still landed in the first array and `secondArray` stayed empty. There is no warning and no way to see it from the outside.

*Symptom:* A consumer who clears or swaps the history by re-pointing `ctrl.sink = []` sees an empty list in the template while copies keep accumulating in an array nothing renders — and `max` silently keeps evicting from the invisible one.

### 5. `clampMax`'s NaN guard cannot catch the NaN it produces; `max` from an `<input>` either collapses the history to one row or removes the cap entirely  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/resolve.ts:66**

`if (value == null || Number.isNaN(value)) return fallback; return Math.max(1, Math.floor(value))`. The guard only catches a literal `NaN`; `Math.floor` of a non-number is where NaN actually appears. Reproduced: `max: 'lots'` → `r.max = NaN` → `sink.length > NaN` is false → the cap never runs (15 entries and counting). `max: ''` → `Math.floor('') = 0` → clamps to 1 and a 6-entry history is spliced down to one row on the next copy. This is not hypothetical: 13-history-picker.vue:156 binds `v-model.number="picker.max"` to `<input type="number">`, and clearing that field writes `''` — the shipped flagship demo destroys the user's history if they clear the max box. The same coercion hole sits in `resolveFeedback` (`'' <= 0` is true), so 06-feedback.vue's cleared duration input silently turns the copied state off for good.

*Symptom:* Either "my history only ever holds one row" (after touching a number input) or an uncapped array that grows without limit — from a function whose entire job is to stop both.

### 6. A controller bound while `disabled` never receives its API, and the documented `error: 'disabled'` is unreachable  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/directive.ts:28**

`apply()` returns at `if (r.disabled)` *before* `enrichController`, so `ctrl.copy`, `ctrl.clear`, `ctrl.history` and `ctrl.copied` are never installed. Combined with the snapshot problem above, re-enabling does not install them either until something the template reads changes. Reproduced: `reactive({ source: 's', disabled: true })` → `typeof ctrl.copy === 'undefined'`, still undefined after `ctrl.disabled = false` + `nextTick`. README.md:354 advertises `error: 'disabled'` as an observable result, but the only channel that can produce it is `ctrl.copy()` — which does not exist in precisely the case that would return it.

*Symptom:* `ctrl.copy?.()` is a silent no-op forever on a control that starts out disabled, and `{{ ctrl.history.length }}` throws on undefined unless every template site remembers `?.`.

### 7. A config-only binding silently becomes a history sink, and then warns you about a history you never asked for  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/controller.ts:37**

With no `sink` in the config, `ensureControllerHistory` still creates `[]` and `directive.ts:35` assigns it to `r.sink`, so `record()` runs for every copy from every config binding. Reproduced: `v-copy:label="{ source: 'tok' }"` leaves the consumer's object as `['source','history','copied','last','copy','clear']` with `history: ['tok']`, and prints `[v-copy] \`key\`/argument is ignored for string history — add the \`.rich\` modifier to keep labels.` — advice about a string history that does not exist. Because `warn.ts:4` latches per message per session, that spurious firing then permanently suppresses the same warning for the consumer's real mistake elsewhere in the app (reproduced: second occurrence printed nothing).

*Symptom:* A console warning telling you to add `.rich` to a binding that has no history, and — worse — total silence later when someone genuinely does put an argument on a string history, because the one-shot was already spent.

### 8. `.once`'s detach is undone by the next re-render, and the README, a demo comment and a test all assert the state that does not survive  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/events.ts:44**

`onTrigger` calls `detachAll` when the latch fires, but nothing in `setupHandlers` consults `state.onceFired`, so the very next `updated` re-attaches the click listener, re-attaches the keydown listener and re-adds `tabindex="0"` + `role="button"`. Reproduced: immediately after the latch both attributes are gone; after one unrelated `{{ tick }}` bump they are back (`tabindex=0 role=button`). The copy is still blocked by `onceFired`, so what remains is a focusable `role="button"` element with live listeners that does nothing — and two dead listeners plus an attribute flicker on every render for the rest of the page's life. README.md:374 ("then every listener detaches (pointer *and* keyboard)") and 08-modifiers.vue:10 state the opposite of the steady state.

*Symptom:* Screen-reader users tab to something announced as a button, press Enter, and nothing happens; a reader who trusts the README reasons about a detached element that is in fact fully wired.

### 9. Keyboard activation is attached for every trigger, so `trigger: 'keydown'` copies twice per Enter  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/events.ts:66**

`wantKeyboard = desired != null && !isNativeInteractive(el)` keys off "a trigger exists" rather than "the trigger is a pointer activation". For any key-shaped trigger both listeners end up on the same element for the same event name. Reproduced: `{ source: 'k', trigger: 'keydown' }` on a span → one Enter → two `writeText` calls, two `copy-result` events, two callback runs (the history hides it because dedupe collapses the rows).

*Symptom:* Double-fired `onSuccess`/`copy-result` — two toasts per copy, two analytics events, two rich history rows once dedupe is off — with no visible cause on the element.

### 10. The shared aria-live region is cached by reference with no connectedness check  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/announce.ts:9**

`if (!liveRegion)` is the only guard; the module keeps the node forever. If anything detaches it — an app that re-mounts its root, a test harness or HMR clearing `document.body`, a "clean up stray nodes" pass — every later announcement is written into an orphan node. Reproduced: after `region.remove()`, subsequent copies put "Copied" on the detached node and no `[aria-live]` element exists in the document. A second-order flake: the package's own tests depend on this cache surviving across test cases (`vCopy.test.ts:151`, `:866` reach for the node by selector and write a sentinel into it).

*Symptom:* Screen-reader announcements stop for the rest of the session after an unrelated DOM cleanup, with nothing logged and nothing visibly different.

### 11. `CopyResult.via` reports `'exec-command'` for copies where no strategy ran  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/execute.ts:47**

The disabled return (`:47`), both refusals (`:94`) and the SSR branch (`clipboard.ts:33`) all stamp `via: 'exec-command'`, while `types.ts:16-18` documents the field as "Which strategy ran" and README.md:435 as "tells you which path ran". The value is a filler constant that happens to typecheck, not a measurement.

*Symptom:* Telemetry that reports a wave of legacy-`execCommand` copies on modern browsers — the exact signal someone would use to decide whether the fallback can be dropped — when those events never touched the clipboard at all.

### 12. README's install block installs a package that does not exist  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-copy/README.md:98**

`npm view @ozjsey/v-copy` → E404. The README opens with an npm version badge linked to a 404 page and `npm install @ozjsey/v-copy` in the Install section, plus a "Part of a set" table where the npm column marks only v-fit-children as published. The manifest admits the real status ("1.1.0 — renamed to @ozjsey/v-copy, ready to publish"); the README does not.

*Symptom:* The first command a stranger runs fails with E404, and the badge makes it look like a published package that was unpublished — worse than admitting it is unreleased.

### 13. "The binding reflects the current selection at copy time" — a string/number binding is a render snapshot  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-copy/README.md:389**

`resolveText` returns `String(r.source)` for a string source; `r.source` was captured at the last render. Only the `TEXT_CONTENT` path and the `() => string` getter form are read at copy time. The aggregated-copy recipe (README.md:389) and 12-multi-select.vue:35 + :89 ("read fresh at copy time") teach the snapshot form with the live-form's guarantee. It is invisible in the demo because the checkbox that changes the selection also re-renders, which is exactly why a consumer will copy the idiom into a place where it is not true.

*Symptom:* A "Copy N selected" button that pastes the previous selection when the source computed depends on something that did not trigger a re-render (a shallowRef, an external store, a non-reactive cache).

### 14. The text-resolution ternary in `executeCopy` contains an arm that can never be observed  ·  `readability`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/execute.ts:58**

`text` is computed with `: r.pending ? '' : resolveText(el, r)`, and the very next statement is `if (override == null && r.pending) return refuse(el, r, 'pending')`. The `''` arm exists only to give the expression a type; the value is always discarded. A reader of this file — the primary distribution channel — has to prove the unreachability themselves before they can touch either line, and a later edit that reorders those two statements turns a refusal into a silent empty copy.

*Symptom:* The next person to edit the refusal order reintroduces the empty-clipboard bug the module's 13-line header comment exists to prevent.

### 15. The entry barrel duplicates the whole public export list with nothing enforcing parity  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-copy/vCopy.ts:10**

`vCopy.ts` re-lists all 13 type names that `src/index.ts` already lists, and `dist/vCopy.d.ts` is generated from `vCopy.ts` only. A type added to `src/index.ts` and forgotten here disappears from the published types with no error — the build succeeds, the tests import from `./vCopy` but assert no export set, and `tsconfig.json` only `include`s `vCopy.ts` so the test file is never type-checked at all (there is no `typecheck` script).

*Symptom:* A consumer's `import type { X } from '@ozjsey/v-copy'` fails for a type the source clearly exports, and nothing in CI notices.

### 16. Dead public and internal surface  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-copy/src/resolve.ts:36**

`export type MutableController = CopyController` is an alias identical to its target whose name claims a mutability distinction the type does not express — it reads as a safety boundary and is none. `clampMax` (resolve.ts:66) and `isBrowser` (clipboard.ts:7) are exported but used only inside their own module and by no test. `CopyEventDetail` (types.ts:31) is part of the published surface, used nowhere in the code, absent from the README's own "all public types are exported" list, while the README's event example uses `CustomEvent<CopyResult>` instead.

*Symptom:* A reader pasting `src/` into their project spends time working out what `MutableController` protects (nothing) and carries two exports nothing calls.

## Comments and docs the code contradicts

- /Users/ozgurseyidoglu/Development/npm/v-copy/README.md:374 — ".once | The trigger fires at most once, then every listener detaches (pointer and keyboard)": the next re-render re-attaches both listeners and re-adds tabindex/role (reproduced).

- /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-copy/08-modifiers.vue:10 — ".once — every listener detaches after the trigger fires once": same lie, in a file a stranger will paste.

- /Users/ozgurseyidoglu/Development/npm/v-copy/README.md:389 and /Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-copy/12-multi-select.vue:35,:89 — "reflects the current selection at copy time" / "read fresh at copy time": a string binding is frozen at the last render; only textContent and the getter form are live.

- /Users/ozgurseyidoglu/Development/npm/v-copy/src/types.ts:152 — "/** The most recent entry. */" on `last`: it is the most recent entry copied *through this particular object*, and goes stale whenever another binding writes to the same sink.

- /Users/ozgurseyidoglu/Development/npm/v-copy/src/controller.ts:34 — "Ensure the controller has a history array; return the (reactive) array to mutate" with `: CopyEntry[]`: returns `undefined` when the write is refused (readonly/frozen), which is what crashes `enrichController`.

- /Users/ozgurseyidoglu/Development/npm/v-copy/src/types.ts:17 and README.md:435 — `via` "Which strategy ran": reports 'exec-command' for disabled bindings, both refusals, and SSR, where nothing ran.

- /Users/ozgurseyidoglu/Development/npm/v-copy/src/history.ts:45 — "`key`/argument is ignored for string history — add the `.rich` modifier to keep labels": fires for config bindings that have no history at all, because one was invented for them.

- /Users/ozgurseyidoglu/Development/npm/v-copy/README.md:98 and the npm badge at README.md:3 — `npm install @ozjsey/v-copy` 404s; the package is unpublished.

- /Users/ozgurseyidoglu/Development/npm/v-copy/src/events.ts:64 — "Built-in keyboard support for non-native-interactive copyables. Native controls already translate Enter/Space to click, so we skip them (no double-copy)": the double-copy it claims to avoid happens anyway when the trigger is itself a key event.

- /Users/ozgurseyidoglu/Development/npm/v-copy/ARCHITECTURE.md:35 — "`resolve.ts` hands it a pre-resolved predicate so the recording path stays branch-free": the recording path in history.ts:39-72 has eight branches, six of them inside the dedupe loop.

## ARCHITECTURE.md claims nothing enforces

- ARCHITECTURE.md:30 — "`history.ts` is the only module that decides what the sink contains": violated three ways. `controller.ts:37` chooses which array the sink *is*, `directive.ts:35` overwrites the resolved `r.sink` with it, and `controller.ts:69` (`clear()`) splices the sink's contents from outside history.ts.

- ARCHITECTURE.md:38 — "`execute.ts` is the only place a copy happens": true today purely by convention. `clipboard.ts` exports `runCopy` to the whole package, and nothing (no lint boundary, no test, no module-private marker) stops the next feature from calling it directly and skipping the two refusals the comment exists to protect.

- ARCHITECTURE.md:4 — "dependencies point strictly downward — no cycles": currently holds (I checked every import), but nothing verifies it; `state.ts` already imports types back out of `resolve.ts`, so the first non-type import in that direction closes a cycle silently.

- ARCHITECTURE.md:47 — "every file under `src/` plus the entry is self-contained TypeScript … take the folder as-is": true of the imports, but the entry duplicates the public export list (`vCopy.ts:11-26` vs `src/index.ts:8-23`) and nothing checks parity, so the folder and the published types can disagree.

- instructions/CONVENTIONS.md:40 — "No defensive programming for impossible states. Trust the types.": `clampMax`'s `Number.isNaN(value)` branch defends against a state `max?: number` already excludes, while failing to handle the NaN the function itself creates one line later. The guard is both unnecessary and wrong.

- instructions/CONVENTIONS.md ("feels native" #5) — "CSS state hooks via `data-<directive>-state` attributes": v-copy ships `[data-copied]` instead, diverging from `v-fit-children`'s `data-fit-children-state`. No rule enforces the shape, so each package picks its own.

- instructions/CONVENTIONS.md:56 requires a per-package strategic brief, and CLAUDE.md calls `instructions/<package>.md` "the source of truth for intent" — there is no `instructions/v-copy.md`. The intent behind the config/controller conflation is recorded nowhere.

## Tests passing for the wrong reason

- vCopy.test.ts:905 '.once detaches the keyboard path too, not just the pointer one' — asserts `tabindex`/`role` are absent in the single tick between the latch and the next patch. One unrelated re-render puts both back (reproduced), so the test pins a transient state and the production steady state is untested.

- vCopy.test.ts:511-514 — the test's own comment admits it: "warnOnce latches per module, not per test — this is the only place the label-dropped path runs, so the spy is guaranteed to see the first firing." The assertion depends on no other test ever touching that message, and on file ordering. There is no reset hook for the `warned` Set.

- vCopy.test.ts:26-28 `flush()` drains exactly six microtasks — green because the current pipeline has fewer than six awaits, not because anything is sequenced. Adding one `await` anywhere in executeCopy silently changes which assertions are evaluated before the copy lands.

- vCopy.test.ts:143-161 and :857-874 — 'announces nothing' is proved with a real 20 ms `setTimeout` plus a sentinel written into a module-cached DOM node that deliberately survives between tests. Both the sleep and the cross-test leakage are load-bearing; the second test even documents the race it is dodging ("otherwise a stray warm-up rAF would clobber the sentinel").

- vCopy.test.ts:532-562 (`scope: 'key'`) binds a plain non-reactive object from `setup`, so `updated` never runs and the controller is enriched exactly once. The playground's equivalent (15-dedupe-scope.vue:25) binds a `computed` whose identity changes on every option flip, driving deregister/re-register and a fresh runtime each time — that path has no test.

- No test asserts that a config-form binding does NOT create a history, or that the consumer's config object is left alone. Both would fail today: the directive adds `history`, `copied`, `last`, `copy`, `clear` to every object binding.

- No test covers the controller's config half changing after mount (`ctrl.disabled`, `ctrl.trigger`, `ctrl.max`, `ctrl.sink`). Every existing controller test sets options before mount, which is exactly the case where the snapshot design looks correct.

- vCopy.test.ts:687 'a plain array sink gets no controller members' tests the one binding form that was never at risk, while the form that does get silently enriched (a config object) is asserted nowhere.
