# COPY-8 — the `v-copy` test harness is sequenced by guesswork, not by signals

P2. Quality-audit "tests passing for the wrong reason", items 3 and 4. **Deferred from 1.1.1** —
it touches all 97 tests' timing, and doing it in the same change as a behaviour fix would make a
regression impossible to attribute. One of the four items *was* fixed in 1.1.1 (see below), which
is why this ticket is narrower than the audit's list.

## Fixed in 1.1.1 — do not re-do

- `warnOnce` latching across tests. `src/warn.ts` now exports an internal `resetWarnings()` (not
  part of the public surface) and `vCopy.test.ts` calls it in `beforeEach`. The label-dropped test
  no longer depends on file ordering, and the new "does not warn about a history it never asked
  for" assertion cannot pass because an earlier test spent the one-shot. A positive control
  ("still warns when a `key` really is dropped by a string history") sits beside it.

## Still open

**1. `flush()` drains exactly six microtasks** (`vCopy.test.ts:26-28`).

```ts
async function flush() { for (let i = 0; i < 6; i++) await Promise.resolve() }
```

Green because the current pipeline has fewer than six awaits, not because anything is sequenced.
Add one `await` anywhere in `executeCopy` and some assertions silently start running before the
copy lands. Replace with a real signal — `vi.waitFor` on the observable the test is about, or a
`copy-result` promise (`onceResult` already exists in the file and is the right shape).

**2. The two "announces nothing" tests** (`:143-161`, `:857-874`) prove a negative with a real
20 ms `setTimeout` plus a sentinel written into `announce.ts`'s module-cached DOM node, which is
deliberately left connected between tests. Both the sleep and the cross-test leakage are
load-bearing, and the second test documents the race it is dodging ("otherwise a stray warm-up rAF
would clobber the sentinel"). 1.1.1 made the region re-create itself when it is disconnected, which
makes a per-test reset possible without breaking the cache: remove the node in `afterEach` and let
the next announcement build a fresh one.

**3. The `scope: 'key'` tests bind a plain object from `setup`** (`:532-562`), so `updated` never
re-resolves. The playground's equivalent (`15-dedupe-scope.vue`) binds a `computed` whose identity
changes on every flip, driving deregister/re-register and a fresh runtime each time. That path has
no unit test — add one that swaps the bound config object identity between copies.

## Acceptance

- No `for (let i = 0; i < N; i++) await Promise.resolve()` left in the file.
- No test depends on state another test left in `document.body`.
- 97+ tests still green, and each changed test fails when the behaviour it names is reverted.
