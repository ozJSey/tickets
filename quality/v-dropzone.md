# Quality audit — `v-dropzone`

**Verdict: structurally-unsound**  ·  13 findings

**PUBLISHED ON NPM — defects here are live for real consumers.**


## The single worst thing

The upload state machine has three unreconciled sources of truth for one question — "what is this zone doing right now?" — and they silently disagree. `instance.uploadBatch` is a single mutable counter object that any new drop overwrites; `instance.records` is a map that is never pruned when a later batch succeeds; and `data-dropzone` on the consumer's own DOM node is read back as the machine's memory (three separate transitions branch on `el.getAttribute('data-dropzone')`). Every state bug in this package, including the two repaired in Run 20, is one of those three disagreeing with the other two — and the worst of them is live right now: drop files, then drop more before the first set finishes, and the zone reports `success` with two uploads still on the wire, then swallows their actual failure entirely (`data-dropzone` stays `success`, forever). Nothing in the code, the tests, or ARCHITECTURE.md ties the three stores together, so each fix to one of them creates the next bug in another. The module split, the naming and the prose are genuinely good; the subsystem they organise is not, and it is the subsystem the package exists for.


## Findings

### 1. A second drop during an in-flight upload overwrites the batch counter, producing a false `success` and then swallowing the real failure  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/upload.ts:330**

`startUploadsForRecords` does `instance.uploadBatch = { total, done: 0, errors: 0 }` unconditionally, discarding the counters of any batch still in flight — and `instance.progressBatch = records.slice()` likewise discards the previous snapshot. The in-flight XHRs from batch 1 are still holding closures that call `advanceBatch`, which now advances the NEW batch's counter. The decision 'is the upload finished?' is therefore fed by a counter that does not describe the set of requests actually outstanding; it should be derived from `instance.records` (the only thing that knows). Confirmed empirically: drop a.png+b.png, drop c.png+d.png while the first two are open, complete only a and b → `data-dropzone` = `success`, `--dropzone-files-pending` = 2, `--dropzone-progress` = 0. Worse, `uploadBatch` is then null, so when c or d actually 500s, `advanceBatch` returns at `if (!batch) return` and the attribute stays `success` — the error never reaches the state machine at all. Playground demo 09 (`/api/upload-slow`, 4s) literally invites this: 'Drop a few files' on a 4-second endpoint, then drop a few more. Untested: the nearest test (vDropzone.test.ts:4619) sets `autoUpload: false` so the second drop only queues and never starts a second batch.

*Symptom:* Zone flashes green 'success' and the progress bar resets to 0 / vanishes while files are still uploading; a file that fails in the second group leaves the zone reading `success` with `api.failed` populated — a user is told their upload worked when it did not.

### 2. A drag gesture resurrects an error the state machine already moved past, because failed records are never pruned by a later batch  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/state.ts:192**

`nonDragRestState` decides 'what is the zone when no drag is in progress' by scanning `instance.records` for any record with status `failed`. Nothing ever removes a failed record on a subsequent drop: records are deleted only on that record's own success, on `cancel`, on `dismissError`, or on unmount. So after one failure and one later success, the attribute correctly goes `success` → `idle` while the failed record is still sitting in the map. The next bare `dragenter`/`dragleave` — a user dragging something over the zone and changing their mind — calls `nonDragRestState`, finds the stale record, and sets `data-dropzone="error"`. Confirmed empirically (state on dragleave: `error`, `api.failed` = ['bad.png'] long after the success). This is the inverse of the Run-20 repair: that run changed this function's input from `uploadBatch` to `records` to stop a drag clearing a sticky error, and the same function now lies in the other direction because the new input is never reset. `startUploadsForRecords`' own comment claims it 'clears a sticky error from the prior batch'; it calls `clearSuccessTimer` and nothing else.

*Symptom:* The zone turns red and the consumer's 'Dismiss' button re-enables out of nowhere, minutes after a successful upload, triggered only by a drag passing over the zone. `api.failed` also accumulates every file that ever failed, so a 'Retry failed (N)' button counts files the user already re-uploaded.

### 3. A throwing `url` / `headers` / `formDataExtras` function escapes as an uncaught exception and strands the zone in `uploading` with no `onError`  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/upload.ts:125**

`resolveVal(config.headers, primary)`, `resolveVal(config.url, primary)` and `buildFormData` (inside `xhr.send(...)`) all invoke consumer-supplied functions with no try/catch anywhere on the URL path, unlike the function-transport path which wraps everything. A throw propagates out of `sendUpload` → `startUploadsForRecords` → `startUploads` → `processFiles` → the drop listener, where it becomes an uncaught error. By then `setState(el, instance, 'uploading')` has already run and the records are already marked `uploading`, so: the zone is stuck at `uploading` forever, `onError` is never called, `api.uploading` permanently lists the file, and the remaining iterations of the per-file `for (const group of groups)` loop never run — so in a multi-file drop the files after the failing one are never sent either. Confirmed empirically (state: `uploading`, onError calls: 0, uncaught `Error: token refresh failed`). The README's own recipe 3 and playground demo 05 both use the function form of `headers` for a token — a token-refresh function is exactly the thing that throws in production.

*Symptom:* An expired-token or presign failure leaves a permanently spinning dropzone with no error message and no way back except a remount; the user retries by dropping again and the zone accumulates more wedged files.

### 4. `releasePickerHost` removes a `position` the consumer wrote, not the one the directive wrote  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/anchor.ts:98**

`pickerHostPositioned` records that the directive wrote `position: relative` at some point — not that the current inline value is still the directive's. `anchorPickerHost` returns early when the host is already positioned and leaves the flag set, so once a consumer positions the host themselves (an object `:style` binding, a conditional class turning into an inline style, a `sticky` header zone), the flag still says 'mine'. The next `teardownClickToPick` — reactive `clickToPick: false`, or `enabled: false`, both of which call it — then does `el.style.removeProperty('position')` and deletes the consumer's value. Vue will not put it back: `patchStyle` skips when the binding value is unchanged. Confirmed empirically: mount → directive writes `relative`; consumer binds `{ position: 'sticky' }`; `clickToPick: false` → inline position is `""`. ARCHITECTURE.md line 40 asserts the opposite: '`pickerHostPositioned` is what stops it reverting a value it did not write.' The only test for this (vDropzone.test.ts:1450) covers the host that arrived positioned at mount — the case the flag genuinely protects — which is why it reads as covered.

*Symptom:* A sticky or relatively-positioned dropzone silently loses its positioning — children fly to the wrong containing block, a sticky header stops sticking — the moment click-to-pick or `enabled` is toggled, with nothing in the console.

### 5. `records` and `progressBatch` retain every failed File for the component's lifetime  ·  `leak`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/state.ts:91**

A failed record is removed only by its own successful retry, `cancel`, `dismissError`, or unmount; a subsequent drop never prunes it (see the error-resurrection finding). `progressBatch` is a second hard reference to the same `FileRecord`s and is cleared only on a transition to `'idle'` — which the sticky `error` state by definition never reaches until someone calls `dismissError()`. So on a long-lived page (the upload widget in an SPA shell) every `File` the user ever failed to upload — and its backing blob — stays reachable from the WeakMap entry for as long as the host element lives. Consumers who never wire a Dismiss button, which the README presents as optional, never release any of it. Note that `progressBatch` holding records already deleted from `instance.records` is deliberate and documented; the leak is that nothing bounds either collection.

*Symptom:* Memory climbs across a session of retried uploads on a flaky connection; an upload page left open grows by the size of every failed file, and no cleanup path runs until the component unmounts.

### 6. `maxCount` is documented as a total and implemented as a per-drop limit  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/validate.ts:47**

`const overCount = typeof opts.maxCount === 'number' && files.length > opts.maxCount` compares only the current event's file list; nothing consults `instance.records` or anything the zone already holds. README:466 and types.ts:195 both say 'Total file count cap', and README:315 calls it 'drop-level' only in the context of which files get rejected. Confirmed empirically: with `maxCount: 2`, two successive two-file drops accept all four files and fire zero `onReject`. The same applies to `multiple: false`, which permits an unlimited number of single-file drops. Under `autoUpload: false` — the mode the README's recipe 8 builds a review queue with — this means `api.pending` grows past `maxCount` without limit and the consumer's server-side cap is the only thing left enforcing it.

*Symptom:* A `maxCount: 5` zone quietly accepts 40 files if the user drops them five at a time; the backend rejects the batch and it looks like the validation option is broken rather than scoped differently than documented.

### 7. `api.cancel(file)` is a silent no-op on a queued file, so a review-before-upload queue has no remove  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/upload-control.ts:16**

`cancelRecordGroup` opens with `if (anchor.status !== 'uploading') return false`, and `api.cancel(file)` turns that false into a bare `return` after a no-op `syncApiArrays`. A file sitting in `api.pending` under `autoUpload: false` therefore cannot be removed through any public API: `cancel(f)` does nothing, `retry(f)` requires `failed`, `dismissError()` only drops `failed`. Confirmed empirically (pending still ['a.png'] after `cancel(f)`). The type docs promise more than this — 'Cancelled files are removed from tracking' (types.ts:156) with no status qualifier, and README:502 says 'cancels just that file's group'. The package's own TASKS.md has this as an open undecided item, so it is known; what is not tracked is that the README already documents the unqualified behaviour.

*Symptom:* The obvious 'remove from queue' button next to each pending file in recipe 8 / demo 08 does nothing at all, and the consumer has no other lever — they end up rebuilding the queue outside the directive.

### 8. `UploadProgressEvent` and `UploadResult` are exported, README-documented public types the library cannot produce  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/types.ts:106**

Both are re-exported from `src/index.ts` (lines 22-23) and listed in README's 'Every consumer-facing type is exported by name' block (lines 555-556), but nothing in the package constructs or accepts either: `onProgress` is `(file: File, percent: number) => void`, and no callback or return value is an `UploadResult`. `UploadProgressEvent` is worse than merely dead — its `{ file, loaded, total, percent }` shape is exactly what a reader assumes `onProgress` receives, so the first thing a consumer writes is `onProgress: (e: UploadProgressEvent) => …`, which does not compile, or they go looking for `loaded`/`total` that the directive never passes on. TASKS.md flags the types themselves as undecided but not the fact that the README already advertises them as usable.

*Symptom:* A consumer types their progress handler from the exported type and gets a compile error or reaches for byte counts that do not exist; deleting the types after publish becomes a breaking change.

### 9. `api.ts` imports `stateMap` and never uses it, and nothing in the build would ever notice  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/api.ts:12**

`stateMap` is in api.ts's import list from './state' and appears nowhere in the file body. tsconfig.json has `strict: true` but no `noUnusedLocals` / `noUnusedParameters`, there is no ESLint config in the package, and `tsup --dts` does not fail on it — so the dead import is invisible to `npm test`, `npm run build` and CI alike. In a package whose stated distribution model is 'people copy the files', a stray import of the module-global WeakMap into the api module also misleads: it suggests api.ts participates in instance lookup, which it deliberately does not (every api method closes over its own `instance`).

*Symptom:* A copy-paste consumer with `noUnusedLocals` on (the common default in new Vue/Vite projects) gets a build error on a file they copied verbatim from a package that claims to be self-contained TypeScript.

### 10. The entire 330-test suite exercises the drop path Chrome does not take, and takes it synchronously  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/vDropzone.test.ts:32**

`makeDataTransfer` builds items with `{ kind, type, getAsFile }` and no `webkitGetAsEntry`, so `gatherDropEntries` returns `null` and every drop test takes the synchronous `extractFiles` branch. In Chromium, an ordinary file drop DOES expose `webkitGetAsEntry`, so production takes `walkEntries(...).then(...)` and `processFiles` runs a microtask later — a different code path and a different timing for every single one of the ~200 drop-driven assertions. PROGRESS.md already records one shipped bug (the null-entry fallback) that this blind spot hid. The consequence today is that nothing in the suite pins the ordering the real browser uses: `drop` sets `dragDepth = 0` and leaves the zone `active` until the walk resolves, `stateMap.get(el) !== instance` is the only re-entrancy guard, and a second drop landing during a walk is entirely unmodelled. The folder describe block at line 5902 is the only place entries exist at all, and it does not re-run the plain-file assertions through that path.

*Symptom:* A regression in the async delivery path — wrong state during the walk, two interleaved walks, a drop after teardown — ships green, because 330 passing tests are all measuring the Firefox/synthetic-DataTransfer branch.

### 11. The FileRecord create-or-reset block is duplicated verbatim in two modules  ·  `readability`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/process.ts:62**

process.ts lines 62-83 and upload.ts lines 346-369 contain the same six-field reset and the same eight-field record literal, character for character apart from the surrounding loop. `FileRecord` has eight fields and two unrelated modules know how to build one; adding a ninth means remembering both sites, and a divergence between them is exactly the class of bug (a record that starts life in the wrong shape depending on whether `autoUpload` was false) that no test would catch because each path is tested separately. For a package whose README tells readers to copy `src/` wholesale, the duplication also denies them the obvious single function — `ensureRecord(instance, file)` — and leaves them guessing which copy is canonical.

*Symptom:* The next field added to `FileRecord` is initialised on one path and left `undefined` on the other; the symptom appears only in the `autoUpload: false` flow and is traced to the wrong module.

### 12. All four drag listeners call `stopPropagation()` unconditionally, and nothing documents it  ·  `readability`

**/Users/ozgurseyidoglu/Development/npm/v-dropzone/src/drag.ts:20**

`dragenter`, `dragover`, `dragleave` and `drop` each call `event.stopPropagation()`. The README documents the `preventDefault` on `dragover` at length (line 597) and never mentions propagation at all, in 718 lines including a 26-item Behavior list and a 10-item Caveats list. The effect is real and invisible: any ancestor's `@dragover`/`@drop` handler never fires, and an enclosing `v-dropzone` loses the `dragenter` bubbling up from its inner zone — which is precisely the +1 its depth counter needs to stay balanced while the pointer crosses a child, so the outer zone's `active` styling drops off while the drag is over the inner one. A reader auditing this file cannot tell whether the `stopPropagation` is load-bearing or copied in alongside the `preventDefault` that is.

*Symptom:* A page-level 'drop anywhere' overlay, or a dropzone nested inside a larger drop target, goes dead or flickers with no explanation, and the consumer reaches for `.capture` modifiers before suspecting the directive.

### 13. The playground's flagship opt-out demo ships a JSON.stringify-diffed MutationObserver as the copy-paste example  ·  `readability`

**/Users/ozgurseyidoglu/Development/npm/playground/src/demos/v-dropzone/13-click-opt-out.vue:63**

`refresh()` serialises two probe objects and compares strings to decide whether to assign state, and its 12-line docblock explains that the guard exists because a MutationObserver on the zone used to hang the tab. That is an honest record of an incident, but the demo is also a copy-paste example on a page whose stated purpose is to show consumers how to use the directive — and what it models is defensive scaffolding around a library hazard, in the card a reader opens to learn what `clickToPick: false` does. Half the file (the `Probe` type, `probe()`, the observer, the `openViaApi` re-probe) is instrumentation for reading the directive's own writes back out of the DOM, which README:277 explicitly tells consumers not to do ('consumer never reads from the DOM'). A stranger pastes the observer pattern, not the directive usage.

*Symptom:* Consumers copy a `JSON.stringify` deep-compare and a subtree MutationObserver into their app as the blessed way to observe a dropzone, then hit the same microtask loop on any other attribute write.

## Comments and docs the code contradicts

- /Users/ozgurseyidoglu/Development/npm/v-dropzone/src/upload.ts:298 — "New upload batch invalidates any pending success → idle clear and clears a sticky error from the prior batch (so the user can see we've moved on)." The line below it calls `clearSuccessTimer(instance)` and nothing else. No sticky error is cleared and no failed record is pruned — this is the comment that makes the error-resurrection bug read as already handled.

- /Users/ozgurseyidoglu/Development/npm/v-dropzone/src/upload-control.ts:94 — "Used by `detach()` on unmount and by `dismissError()` after error." `dismissError()` (src/api.ts:90) does not call `abortAllUploads`; it filters failed entries out of the map by hand. If it did call it, the README's promise that `dismissError()` transitions to "`uploading` if some files are still in flight" would be impossible, since `abortAllUploads` aborts and clears everything.

- /Users/ozgurseyidoglu/Development/npm/v-dropzone/src/state.ts:36 — "in per-file URL mode and in function-based mode, group is always `[this]`." Both record factories (process.ts:76, upload.ts:362) create `group: []`, and it stays empty for the whole life of a queued record. That is why four call sites in upload-control.ts (lines 18, 142, 159, 169) carry a `group.length > 0 ? group : [anchor]` fallback the doc says is unnecessary — a defensive branch for a state the doc claims the types exclude.

- /Users/ozgurseyidoglu/Development/npm/v-dropzone/README.md:423 and :603 — "`error` … Sticky — never auto-clears. Cleared on the next drop" / "The next drop reseeds the state machine." The attribute moves off `error`; the failed records that define `error` are never cleared by a drop, so a later dragenter/dragleave puts it straight back (confirmed).

- /Users/ozgurseyidoglu/Development/npm/v-dropzone/README.md:466 and src/types.ts:195 — "Total file count cap." It is a per-event cap; two drops of N each both pass a `maxCount: N`.

- /Users/ozgurseyidoglu/Development/npm/v-dropzone/README.md:555-556 — `UploadProgressEvent` and `UploadResult` listed under "Every consumer-facing type is exported by name". Neither type is produced or accepted anywhere in the package; TASKS.md calls them "unreachable by construction".

- /Users/ozgurseyidoglu/Development/npm/v-dropzone/README.md:434 — "Both vars are set when uploads start … and clear when the host returns to idle (cancel, success auto-clear, or unmount)." A second drop during an in-flight batch replaces `progressBatch` outright, so the vars stop describing the files in flight well before any of those three events (measured: `--dropzone-progress` 0 with two files at 100% and two at 0%).

## ARCHITECTURE.md claims nothing enforces

- ARCHITECTURE.md:59 'Every write into the consumer's host is idempotent.' Only `syncPickerAttrs` and `applyPickerA11y` were made idempotent (via `setAttr`). `setState` (src/state.ts:136) calls `el.setAttribute('data-dropzone', state)` unconditionally, and `processFiles` calls `setState(…, 'idle')` on every empty pick and every non-upload drop — writing the same value and queuing a MutationRecord every time. The rule as written is false; what is true is narrower ('the two picker syncs are idempotent'), and nothing prevents the next module from adding a third unconditional write.

- ARCHITECTURE.md:68 '`state.ts` owns every reflection. Nothing else writes `data-dropzone`, the CSS variables, or the api arrays directly.' src/api.ts:134 writes `instance.api.state` directly, and it sources the value by reading `data-dropzone` back off the element — a reflection target being used as the state store. Three other sites (process.ts:51, upload.ts:82, api.ts:97) branch on `el.getAttribute('data-dropzone')`, so the consumer's DOM node is load-bearing memory for the state machine, not a projection of it. Nothing enforces the rule: `el.setAttribute` and `el.style.setProperty` are reachable from every module.

- ARCHITECTURE.md:40 '`pickerHostPositioned` is what stops it reverting a value it did not write.' The flag records that the directive wrote a position at some point, not that the current inline value is still the directive's, so it does not stop that (see the anchor finding, confirmed empirically).

- ARCHITECTURE.md:3 'dependencies point strictly downward — no cycles.' Currently true (verified by reading every import), but nothing checks it — no lint rule, no dependency-cruiser, no test. With ten interdependent modules and `process.ts` imported by three of them, the first accidental back-edge (e.g. `state.ts` reaching for `writeUploadVars`' caller) ships silently.

- ARCHITECTURE.md:20 '`process.ts` is THE pipeline … validation and state transitions can never diverge between input paths.' Validation cannot diverge, but state transitions already do: `api.upload()` (no arg) and both retry paths call `startUploadsForRecords` directly, and that function is where the batch counter is overwritten. The single-pipeline claim is about files entering, and it is quietly read as a claim about state, which is where the defects are.

## Tests passing for the wrong reason

- vDropzone.test.ts:4619 'a queued drop during an in-flight upload keeps the zone "uploading"' — the only test that drops twice without letting the first batch settle, and it passes `autoUpload: false` so the second drop only queues and never calls `startUploadsForRecords`. It is the safe neighbour of the batch-overwrite defect: its title claims to cover 'a drop during an in-flight upload', and the one variant that breaks (the default `autoUpload: true`) is the variant it avoids.

- vDropzone.test.ts:1450 'leaves a host that already has a position of its own alone' — asserts the host that arrived positioned at mount, which is the one case `pickerHostPositioned` genuinely protects. The failing case (directive anchors a static host, consumer positions it afterwards, teardown deletes the consumer's value) is untested, so the suite reads as pinning the ARCHITECTURE invariant it does not pin.

- vDropzone.test.ts:5700 'a new drop during success state cancels the pending success→idle timer and starts a new batch' — drops during `success`, i.e. after `uploadBatch` is already null, so the overwrite is harmless by construction. The assertion is true for a reason unrelated to the hazard the title suggests it is guarding.

- The whole drop-driven suite (makeDataTransfer, vDropzone.test.ts:32) — items carry no `webkitGetAsEntry`, so every assertion measures the synchronous fallback branch that Chromium does not use for file drops, and measures it synchronously where production defers a microtask. Green here says nothing about the path real users take.

- vDropzone.test.ts:4033 'unmount mid-upload fires onError with `aborted: true`' and the XHR cancel tests — the FakeXHR's `abort()` only sets a flag and never invokes `onabort`, so the tests drive `emitAbort()` by hand. The real browser fires the abort event synchronously from inside `xhr.abort()`, i.e. in the middle of `abortAllUploads`' iteration over `instance.records` and before `records.clear()`. The duplicate-emission guard (`abortAnnounced`) is therefore only ever exercised against an ordering the browser does not produce.
