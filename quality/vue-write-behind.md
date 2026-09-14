# Quality audit — `vue-write-behind`

**Verdict: structurally-unsound**  ·  14 findings

**PUBLISHED ON NPM — defects here are live for real consumers.**

> **Status 2026-09-13 — addressed in `vue-write-behind` 0.1.1 (built, not published).**
> 12 of the 14 findings are fixed in the source, including all six behavioural ones (1, 2, 3,
> 4, 6, 9) and every doc/test/script honesty finding (5, 7, 8, 10, 11, 12). Finding 13 is
> resolved in the direction of honesty — one `Date.now()` call site, `ARCHITECTURE.md`
> corrected — with the injectable clock deferred to `tickets/WBC-6`. Finding 14 (playground tab,
> brief, backlog entry) is deferred to `tickets/WBC-3` and `tickets/WBC-5`.
> Evidence: the data loss is pinned by an end-to-end test asserting what the *server* holds; the
> suite is mutation-tested at 58/59 with the one survivor argued equivalent; `check:dist` and
> `check:browser` both go **red** against a clean build of 0.1.0 and green on 0.1.1. Details in
> `vue-write-behind/CHANGELOG.md`.


## The single worst thing

Flight identity lives ONLY inside the entry that `discard()` deletes, so `discard(key)` on an in-flight key erases the library's only record that a request is in the air — and the next edit sends a second, concurrent request for the same key. I drove it end to end: discard while a 600ms PUT is out, type again, the fast second request lands first, the slow first request lands last, and the server ends up holding the OLD value while the screen shows the new one and `pending` reports `[]` ("all saved"). README.md:117 and ARCHITECTURE.md:44 both state the opposite as an absolute guarantee ("two requests for one cell can never race and land out of order"). There is no observable trace: no error, no failed entry, nothing to trace back to `discard`. Structurally this is the same disease as the `dueAt` field below — the `Entry` record is the single mutable home for several independent facts (is a request out, when may this key next go, why is it waiting), and every operation feels entitled to overwrite all of them at once.


## Findings

### 1. discard() of an in-flight key destroys the one-request-per-key invariant  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/outbox.ts:162**

`discard` is `entries.delete(key)` with no check for `sentVersion !== undefined` and no record kept of the orphaned flight. The outbox's ONLY memory that a request exists is `entry.sentVersion`, which is deleted along with the entry. A subsequent `set(key, …)` creates a fresh entry with `sentVersion: undefined`, and the next `take()` happily claims it. `take()`'s in-flight skip (outbox.ts:122) cannot help — the entry it would have skipped no longer exists. This directly falsifies README.md:117 and ARCHITECTURE.md:44, which state it as a guarantee, not a tendency. The proof is sitting in the test suite: outbox.test.ts:107 builds exactly this two-concurrent-flights situation and asserts only that the LATE response is ignored, never that a second request should not have been sent.

*Symptom:* Verified end to end: cell shows 'v2-fast', outbox.pending is [] (UI says 'all saved'), and the server applied ['v2-fast', 'v1-slow'] in that order — it now holds the stale value permanently. A support ticket that reads 'the spreadsheet shows the right number but the export has the old one', with nothing in the client to trace it to.

### 2. `dueAt` conflates the debounce quiet period with the retry backoff, so settle() and fail() silently cancel the user's debounce  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/outbox.ts:144**

One field means three things: a debounce deadline (written by `set`), a backoff deadline (written by `fail`), and `0` = 'eligible now'. `set` is careful — `Math.max(entry.dueAt, dueAt)` at outbox.ts:104, with a comment explaining why. Nothing else is. `settle`'s superseded branch does a bare `entry.dueAt = 0` (line 144, comment: 'the server is evidently healthy, so drop the backoff') — but it is dropping whatever is in the field, which after an edit made during the flight is the DEBOUNCE window, not a backoff. `fail` does a bare `entry.dueAt = now + delay` (line 158), shortening a debounce longer than the backoff. `clearBackoff` (public `retry()`) zeroes it for every entry (line 172), including keys that are merely mid-typing. Eligibility is therefore decided by a value last written by a response handler rather than by the debounce policy — the decision is fed the wrong input.

*Symptom:* Verified with the real composable: `debounce: 2000`, keystroke at t=2303ms, the write left at t=3041ms — 738ms after the keystroke, because a superseded settle at ~3000ms zeroed the window. Reported as 'I set debounce: 2000 and it still fires while I'm typing', reproducible only when a save is already in flight, which is the normal state for a field being typed into. No test covers debounce combined with an in-flight key.

### 3. `retryAt` publishes the internal `dueAt` sentinel raw — consumers get epoch 0 (1970) and negative countdowns  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/outbox.ts:187**

`retryAt: entry.blocked ? undefined : entry.dueAt` hands the internal field straight out. types.ts:68 documents it as 'Epoch ms of the next automatic attempt'; outbox.ts:41 documents the same field as '`0` = eligible now'. Both cannot be true. Two reachable states break it. (1) `retry: false`, a failure, then the user types again: `set` clears `blocked` but leaves `attempts` at 1 and `dueAt` at 0 → `failures()` emits `retryAt: 0`. (2) A backed-off key that is currently being retried: `take` sets `sentVersion` but leaves `attempts` and `dueAt` alone, so the key appears in `failed` AND `inFlight` simultaneously with a `retryAt` in the past. The tests only ever assert `retryAt` in the two states where it happens to be right (outbox.test.ts:153, :250; useWriteBehind.test.ts:189, :517).

*Symptom:* Verified: after a retry:false failure plus one keystroke, `failed[0].retryAt === 0`, so `new Date(retryAt)` renders 'Jan 1, 1970'. During a retry, `retryAt - Date.now()` is -500, so a 'retrying in Ns' countdown runs negative while the same key is also rendered as syncing.

### 4. flush() cannot send a key blocked by `retry: false`, resolves as if it had, and flushOnHidden goes through flush()  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/flush.ts:90**

`dispatch(true)` passes `Number.POSITIVE_INFINITY` as `now`, which defeats `dueAt` but NOT `entry.blocked` (outbox.ts:123). Under `retry: false` every key that fails once is blocked forever, so `flush()` is inert for exactly the keys most at risk. It still returns a resolved `Promise<void>` — a `flush(): Promise<void>` that resolves identically whether it sent everything, sent nothing, or every request failed gives the caller no way to know. `store.flush` is what the tab-hidden handler calls (useWriteBehind.ts:144), so the library's own last-chance save is the thing that silently does nothing.

*Symptom:* Verified: retry:false, one failure, server comes back, `await wb.flush()` → zero further writer calls, `pending` still ['A1'], promise resolved. A consumer doing `await outbox.flush()` on unload, or just closing the tab, loses the write — in a library whose one-line pitch is 'a failed save never rolls back or drops what the user typed'. types.ts:145-148 and README.md:61 both say flush ignores 'the debounce and backoff clocks'; README.md:56 defines `pending` as including in-flight keys, so 'send everything pending' is doubly wrong.

### 5. The Vue-level H5 test passes with the guard it claims to test deleted  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/useWriteBehind.test.ts:155**

'never has two requests in flight for one key (H5)' edits the key three times while a request is out and asserts `net.calls` stays at 1. But `hasScheduledWork()` (outbox.ts:195) returns false while the only entry is in flight, so `onOutboxChange` calls `scheduler.stop()` and no timer runs for the whole loop — `dispatch()` is never invoked and `take()` is never called. I built a bundle with `if (entry.sentVersion !== undefined) continue` removed from `take()` and ran the test's exact shape: identical output (calls=1, inFlight=['A1'], then ['first','fourth']). The mutant survives. The same shape sits at useWriteBehind.test.ts:244 ('does not fire a request per keystroke while an endpoint is failing'), where the loop advances 4×100ms against a 1000ms interval — the timer provably cannot fire, so removing `entry.dueAt = now + delay` from `fail()` would not fail it either.

*Symptom:* Two of the invariants the README states as absolutes have Vue-level tests that would not notice their removal. Someone refactoring `take()` or `fail()` gets a green suite. The outbox-level tests do cover the guards, so the harm is false confidence in the end-to-end layer, which is where the discard hole above actually lives — and no test there catches it.

### 6. The retry backoff curve is quantized to `interval`, so `retry.initialDelay` is inert whenever interval exceeds it  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/scheduler.ts:32**

`createBackoff` computes exact millisecond delays, but the only non-forced caller of `take(now)` is `scheduler.onTick`, which fires on a fixed `interval` grid (useWriteBehind.ts:70). The effective retry time is the next multiple of `interval` at or after `dueAt`. With the documented defaults (interval 1000, initialDelay 1000) they coincide exactly, which is why the tests — scheduler.test.ts:9 tests `createBackoff` in isolation, useWriteBehind.test.ts:209 uses the default interval — never see it. Nothing in README.md:45, types.ts:50-58 or the JSDoc mentions that the curve depends on `interval`.

*Symptom:* Verified: with `retry: { initialDelay: 100, factor: 2 }`, interval 100 gives gaps 101/202/404/810 as advertised; interval 500 gives 503/502/502 — the curve is gone and the option had no effect. A consumer setting `interval: 30000` ('save every 30s') silently gets a flat 30s retry instead of the documented 1→2→4→8→16→30s, and their `retry` config does nothing at all.

### 7. browser-check's headline assertion is a case-sensitivity heuristic coupled to the fixture  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/scripts/browser-check.mjs:130**

The file's header says it exists 'for the one claim no unit test can make: the cell does not jump'. The check is `[...observed].filter((value) => value !== value.toLowerCase())` — it flags a jump only if the input ever contained an uppercase character. It works solely because playground.html:152 happens to answer with `String(value).toUpperCase()`. Change the fixture to echo the value, return a lowercase string, or return '' and the check passes no matter what the library does with the response. It measures casing, not whether the input ever held a value the user did not type. The honest assertion is right there for free: the script knows the exact prefix sequence it typed.

*Symptom:* A stranger copying `src/` and adding response-application (the single most tempting change to make to this library) would get a green BROWSER CHECK. The one script that exists to defend the product's whole claim cannot detect the failure it was written for.

### 8. Both verification scripts are fixed sleeps racing the intervals they measure, and one of them lies in its own output  ·  `flake`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/scripts/browser-check.mjs:106**

browser-check.mjs has nine bare `await sleep(...)` calls against a 1000ms flush interval and a 2000ms fake server; the `waitFor` poll helper it defines at line 91 is used exactly once (line 147). `check('two requests for eleven keystrokes', puts.length === 2)` at line 145 depends on 8 CDP keystrokes plus 8 DOM reads finishing inside the ~1700ms window before the first response lands. dist-check.mjs is worse: `interval: 20` with a 30ms writer and `await sleep(30)` before reading `outbox.inFlight` — ten milliseconds of margin on a shared CI box. And dist-check.mjs:75 prints `ok  ten edits, one window, one request` while asserting `calls.length === 2` from two edits — the label is wrong about the input count, the window count and the request count.

*Symptom:* Intermittent red on a clean tree (the classic 'just re-run it'), which trains everyone to ignore the only two checks that touch the built artifact — in a repo whose CLAUDE.md records dist going stale silently three times. And when someone reads the passing output to learn what is guaranteed, dist-check tells them one request went out when two did.

### 9. syncStore() rebuilds three arrays on every outbox transition, making the README's own persistence recipe fire constantly and bulk edits quadratic  ·  `rot`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/useWriteBehind.ts:124**

`onChange` fires on every single outbox transition (set, take, settle, fail, discard, clearBackoff) and `syncStore` unconditionally allocates fresh `pendingKeys()`, `inFlightKeys()` and `failures()` arrays. Reference identity therefore changes even when the contents do not, so any consumer `watch` on `pending`/`failed` re-fires on transitions that changed nothing they can see. And because `syncFromSource` (line 86) calls `outbox.set` once per changed key, an N-key change costs N full rebuilds — O(N²).

*Symptom:* Verified two ways. (1) The README's own 'No persistence / IndexedDB' snippet (README.md:143-148) performed 9 synchronous localStorage writes in 600ms of retrying while `pending` never changed from ["A1"] — during a sustained outage that is a JSON.stringify + localStorage write on every retry transition, forever. (2) Pasting into the README's own spreadsheet-cell example: 2000 cells = 86ms, 4000 = 244ms, 8000 = 1040ms of blocked main thread for one paste.

### 10. The SSR test and dist-check both document a hand-off that cannot exist  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/vueWriteBehind.ssr.test.ts:44**

'…but the edit is still queued, so a client-side hydrate can send it', and dist-check.mjs:35 `check('server-side: the edit is still queued for the client', …)`. The queue is a `Map` closed over by `createOutbox` inside a server-render's effect scope. Hydration constructs a brand-new composable with a brand-new outbox and cannot see it; nothing serialises it and no API exposes it for serialisation. Worse, on the server the outbox happily accumulates entries for the life of the render with no timer to drain them.

*Symptom:* Someone reads this and believes SSR-side edits survive to the client, and does not build the hand-off they actually need. Both assertions pass, so the belief is reinforced by a green suite.

### 11. README claims Promise.allSettled parallelism; the word appears nowhere in the source  ·  `lie`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/README.md:43**

'Parallelism | `Promise.allSettled` over the due keys'. `flush.ts` uses per-request try/catch in `runPerKey` and `Promise.all([...inAir])` in `flush()`. The behaviour happens to be equivalent for the per-key path, but the README states an implementation that does not exist and `flush()` genuinely uses `Promise.all`. flush.ts:10 at least hedges with 'allSettled semantics'; flush.test.ts:85 puts '(allSettled, not all)' in a test name for code containing neither.

*Symptom:* A reader grepping the source for `allSettled` to understand the failure semantics finds nothing and has to reverse-engineer it. In a package whose stated distribution model is source-copying, a README that describes code that is not there is the defect.

### 12. Dead surface: an unused debug handle with a comment claiming it is used, two methods no production code calls, and a dead vitest config  ·  `dead`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/playground.html:178**

playground.html:178-179 sets `window.__writeBehind` under the comment 'Handles for the headless check in scripts/browser-check.mjs' — browser-check.mjs never reads it (it goes through the DOM); grep finds exactly one occurrence in the repo. `Scheduler.isRunning()` (scheduler.ts:62) and `Outbox.isEmpty()` (outbox.ts:211) have no caller outside their own definitions and the tests. scheduler.test.ts:98 'reports whether it is running' asserts `timer !== undefined` against itself after start/stop — a tautology testing a method nothing uses. `vitest.config.ts` is silently ignored the moment `vitest.workspace.ts` exists.

*Symptom:* Someone maintaining the browser check deletes `window.__writeBehind` and hesitates, or preserves it forever; someone counts scheduler.test.ts's 10 tests as coverage when one of them exercises dead code. Small individually, but this is a package that sells itself on 'read the folder in one sitting'.

### 13. The injected clock in flush.ts is a half-measure — useWriteBehind reads Date.now() inline  ·  `readability`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/useWriteBehind.ts:85**

flush.ts:30 declares `now: () => number` with the comment 'Injected so the flusher owns no clock of its own', and ARCHITECTURE.md sells the whole split on 'every clock reading arrives as an argument'. But useWriteBehind.ts:85 and :111 call `Date.now()` directly to compute the debounce deadline, and the scheduler owns a raw `setInterval`. So the composable cannot be driven by a fake clock through the seam the architecture advertises — every Vue-level test has to reach for `vi.useFakeTimers()` + `vi.setSystemTime`, which is exactly what the injection was supposed to avoid. Two of the three time sources bypass the seam.

*Symptom:* A reader trusts ARCHITECTURE.md's claim, tries to test the debounce path deterministically by passing a clock, and finds there is no way to. It also explains why the debounce-during-flight bug above has no test: the only way to write one is with system-time manipulation the design implies is unnecessary.

### 14. No playground tab, no strategic brief, no entry in the root backlog — and playground.html is not copy-pasteable  ·  `readability`

**/Users/ozgurseyidoglu/Development/npm/vue-write-behind/playground.html:1**

`playground/src/demos/vue-write-behind/` does not exist, nothing in `playground/src` mentions the package, there is no `instructions/vue-write-behind.md`, and grep finds no reference in the root TASKS.md or PROGRESS.md. CLAUDE.md is explicit that a package's tab is part of done and that the shared playground is where behaviour gets verified in a real browser. What exists instead is a 186-line standalone HTML file with an inline import map pointing at `./node_modules/vue/dist/vue.esm-browser.js` and `./dist/vueWriteBehind.min.js` — it is a CDP fixture, not an example. Nothing in it can be pasted into a Vue SFC app, and its own comment (line 134) tells the reader to hand-edit the import map first.

*Symptom:* For a repo whose stated distribution model is 'people copy the source', the only worked examples this package ships are three cards welded to a raw-HTML harness. A stranger who wants a working reference has the 15-line README snippet and nothing else — and the README snippet is the bare form, so nobody ever sees a correct `debounce`, `retry: false`, `keys` or batch usage in context.

## Comments and docs the code contradicts

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/README.md:43 — 'Promise.allSettled over the due keys'. No allSettled anywhere in src/; the per-key path is try/catch and flush() uses Promise.all.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/README.md:45 — 'per-key backoff 1 → 2 → 4 → 8 → 16 → 30 s (capped)'. True only when `interval` divides the delays; with interval 500 and initialDelay 100 the measured gaps are 503/502/502.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/README.md:61 — 'flush() | send everything pending now, ignoring the debounce and backoff clocks'. It skips `blocked` keys (retry:false), and README.md:56 defines `pending` as including in-flight keys, which flush also skips.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/README.md:117 — 'A key already in flight is skipped even when it is dirty, so two requests for one cell can never race and land out of order.' Falsified by discard() on an in-flight key; verified out-of-order landing.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/ARCHITECTURE.md:44 — same claim, stated as a property that 'falls out' of the design. Nothing enforces it.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/types.ts:68 — retryAt: 'Epoch ms of the next automatic attempt'. Emits 0 (1970) for a re-armed retry:false key and a past timestamp while the retry is in flight.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/types.ts:83 — 'An edit never *shortens* an active backoff.' True of `set`, but the doc omits that settle(), fail() and clearBackoff() all shorten (or zero) an active DEBOUNCE, which shares the same field.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/types.ts:146 — flush(): 'Send every pending key that is not already in flight, ignoring the debounce and backoff clocks.' Blocked keys are silently excluded.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/outbox.ts:41 vs src/types.ts:68 — the same field is documented as '`0` = eligible now' internally and as 'epoch ms of the next automatic attempt' publicly. Both cannot hold; the public one loses.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/outbox.ts:139 — 'the server is evidently healthy, so drop the backoff'. It drops whatever is in `dueAt`, which after an edit during the flight is the debounce window.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/flush.ts:10 — '(Promise.allSettled semantics …)'. No allSettled; the sibling-isolation comes from per-request try/catch.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/flush.ts:35 — dispatch: '`force` ignores the debounce and backoff clocks'. It ignores dueAt but not `blocked`.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/src/useWriteBehind.ts:135 — 'never leave one running with nothing to send'. hasScheduledWork() ignores dueAt, so a key in a 30s backoff keeps the 1s interval spinning through 30 no-op ticks.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/vueWriteBehind.ssr.test.ts:44 — '…but the edit is still queued, so a client-side hydrate can send it.' The server outbox is a per-render closure Map; hydration builds a new one and cannot see it.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/dist-check.mjs:35 — 'server-side: the edit is still queued for the client'. Same impossible hand-off, asserted as a passing check.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/dist-check.mjs:75 — prints 'ten edits, one window, one request' while asserting calls.length === 2 from two edits.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/scripts/browser-check.mjs:8 — 'the only honest proof is a real keyboard against a real <input>'. The proof is `value !== value.toLowerCase()`, which only detects a fixture that happens to uppercase.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/flush.test.ts:85 — test name '(allSettled, not all)' for code using neither.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/playground.html:178 — 'Handles for the headless check in scripts/browser-check.mjs.' browser-check.mjs never reads window.__writeBehind.

- /Users/ozgurseyidoglu/Development/npm/vue-write-behind/ARCHITECTURE.md:60 — 'every clock reading arrives as an argument'. useWriteBehind.ts:85 and :111 call Date.now() inline and scheduler.ts owns a raw setInterval.

## ARCHITECTURE.md claims nothing enforces

- ARCHITECTURE.md:44 'A key already in flight is skipped, even when it is dirty, so two requests for one cell can never race and land out of order.' Nothing enforces this across discard(): the entry holding sentVersion is deleted, the outbox forgets the request exists, and the next set()+take() issues a second concurrent request. Verified out-of-order landing leaving the server with the stale value.

- ARCHITECTURE.md:30 'Nothing outside outbox.ts may clear a dirty key.' This one IS genuinely enforced — `entries` is a closure variable and the Outbox interface exposes only settle/fail/discard/clearBackoff. Credit where due. But it is also the least interesting of the three, and its enforcement is being used to imply the other two are equally safe.

- ARCHITECTURE.md:47 'The value is read out of the map at send time (take()), never captured when the edit happened.' Holds inside the outbox, but useWriteBehind.ts keeps a parallel `shadow` Map of last-seen values that IS captured at edit time and is what decides whether an edit exists at all. Nothing keeps the two in step: store.set() writes shadow directly (line 110) while syncFromSource writes it from the watcher, and shadow accumulates keys that `keys` excludes.

- ARCHITECTURE.md:60 'outbox.ts has no Vue, no timers and no I/O — every clock reading arrives as an argument.' True of outbox.ts, false of the split as a whole: useWriteBehind.ts reads Date.now() inline twice for the debounce deadline, so the composable has no clock seam and the debounce path cannot be tested deterministically through the architecture's advertised boundary.

- ARCHITECTURE.md:72 'every file under src/ plus the entry is self-contained TypeScript with no dependency beyond the vue peer — take the folder as-is.' The modules import each other extensionlessly ('./flush', './outbox'), which resolves only under moduleResolution: bundler (what tsconfig.json:5 sets). Copied into a NodeNext or plain Node ESM project the imports do not resolve.

- CLAUDE.md's playground-coverage rule ('a change to a v-* public API is not finished until its tab is updated'). There is no playground/src/demos/vue-write-behind/, no reference to the package anywhere in playground/src, no instructions/vue-write-behind.md brief, and no entry in the root TASKS.md or PROGRESS.md.

## Tests passing for the wrong reason

- useWriteBehind.test.ts:155 'never has two requests in flight for one key (H5)' — passes because hasScheduledWork() stops the scheduler while the key is in flight, so take() is never called. Verified: with take()'s in-flight guard deleted from the bundle, the test's exact sequence produces identical output (calls=1, inFlight=['A1'], then ['first','fourth']). The mutant survives.

- useWriteBehind.test.ts:244 'does not fire a request per keystroke while an endpoint is failing' — the loop advances 4×100ms against a 1000ms interval starting at t=1000, so the timer provably cannot fire before the assertion. Removing `entry.dueAt = now + delay` from fail() would not fail it. It asserts nothing happened in a window where nothing could.

- outbox.test.ts:107 'ignores a response from a flight that is no longer current' — sets up two concurrent live flights for one key via discard()+set()+take() and asserts only that the stale response is ignored. It exercises, and thereby blesses, the exact state ARCHITECTURE.md:44 says can never occur.

- scheduler.test.ts:98 'reports whether it is running' — asserts `timer !== undefined` against itself after start/stop, on a method (Scheduler.isRunning) that no production code calls.

- outbox.test.ts:248 `expect(box.take(Number.POSITIVE_INFINITY)).toEqual([])` — cements as correct the behaviour that makes flush() inert for retry:false keys, while README.md:61 and types.ts:146 promise the opposite.

- useWriteBehind.test.ts:485 'debounce holds a key back until the typing stops' — only ever exercises debounce on a key that is NOT in flight, which is the one case where the dueAt conflation cannot bite. There is no test combining debounce with an in-flight key.

- dist-check.mjs:75-80 — the whole client pass turns on `sleep(30)` against a 20ms interval and a 30ms writer; it passes on a quiet machine and its labels do not describe what it asserts.
