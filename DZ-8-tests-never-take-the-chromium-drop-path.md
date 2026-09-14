# DZ-8 — the unit suite exercises the drop path Chromium does not take

P1. Quality-audit finding 10, plus the FakeXHR abort-ordering note. Deferred from 0.1.1: this is
test-harness work of a size that would have dwarfed the live-bug fix it rode with, and getting it
wrong makes 345 green assertions meaningless rather than merely narrow.

## The gap

`makeDataTransfer` (`vDropzone.test.ts:32`) builds items as `{ kind, type, getAsFile }` with **no
`webkitGetAsEntry`**, so `gatherDropEntries` returns `null` and every drop test takes the
synchronous `extractFiles` branch. In Chromium an ordinary file drop *does* expose
`webkitGetAsEntry`, so production takes `walkEntries(...).then(...)` and `processFiles` runs a
microtask later. That is a different code path **and** a different timing for every one of the
~200 drop-driven assertions.

`PROGRESS.md` already records one shipped bug the blind spot hid (the null-entry fallback). What
is unpinned today: `drop` sets `dragDepth = 0` and leaves the zone `active` until the walk
resolves; `stateMap.get(el) !== instance` is the only re-entrancy guard; and **two interleaved
walks are entirely unmodelled** — which is the same overlapping-input shape as the 0.1.1
batch-counter defect, one layer up.

Second, smaller: the FakeXHR's `abort()` only sets a flag and never invokes `onabort`, so the
abort tests drive `emitAbort()` by hand. A real browser fires the abort event **synchronously
from inside `xhr.abort()`** — i.e. in the middle of `abortAllUploads`' iteration over
`instance.records`, before `records.clear()`. The `abortAnnounced` duplicate-emission guard is
only ever exercised against an ordering the browser does not produce.

## Shape of the fix

- Give `makeDataTransfer` a `webkitGetAsEntry` by default, so the *default* test drop is the
  Chromium drop, and keep a named opt-out helper for the legacy-fallback assertions.
- Re-run the plain-file assertions through the entry path rather than only the folder describe
  block at `vDropzone.test.ts:5902`.
- Add: a second drop landing during an unresolved walk; a drop after teardown mid-walk (the guard
  exists, nothing drives it); the state the zone reports *during* the walk.
- Make FakeXHR's `abort()` fire `onabort` synchronously, and check what that does to
  `abortAllUploads`.

## Acceptance

- Every currently-green drop assertion still passes through the entry path, or the difference is
  explained and pinned as its own test.
- The interleaved-walk case is covered, and fails without a fix if one turns out to be needed.
- The browser-side `pnpm interactions` folder checks stay green — they already drive a real
  directory through `Input.dispatchDragEvent`, which is the only honest source of entries.
