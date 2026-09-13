# WBC-2 — scaffold `vue-write-behind` (phase 1: the package)

Verdict and full design: `DESIGNS.md` → "WBC-1". Standards: `tickets/_STANDARDS.md`.
**Phase 1 is the package only.** The playground tab and documentation view are WBC-3 — but read
`_STANDARDS.md` anyway, because the API you choose here has to be demonstrable there.

## Why it exists (the README lead)

> **The write never comes back.** Local state stays authoritative and the network is a background
> chore. Edit a cell ten times and one request goes out carrying the tenth value. The server's
> reply is *discarded on purpose* — it can never overwrite the cell the user is still typing in.
> A failed save rolls nothing back; the key stays dirty and goes out again next tick, carrying
> whatever has been typed since.

Second line, as the scope fence: **it is a state outbox, not an operation log.** Keys are
independent and last-write-wins.

## Module map (standards item 2 — `ARCHITECTURE.md` required)

- `src/outbox.ts` — pure key/version/dirty state machine. **No Vue, no timers, no I/O.**
- `src/scheduler.ts` — interval + per-key backoff clock
- `src/flush.ts` — `allSettled`-per-key and batched-call adapters
- `src/lifecycle.ts` — `visibilitychange`, `onScopeDispose`, SSR guard
- `src/useWriteBehind.ts` — the Vue surface
- `src/types.ts`, thin `src/index.ts`

**Invariant for `ARCHITECTURE.md`: nothing outside `outbox.ts` may clear a dirty key.**

## The correctness core — get this right first

**H1 — a key edited while its own flush is in flight.** A boolean dirty flag is *wrong*: the
resolving response clears a flag a newer edit set, and that edit is lost forever. Use a
**monotonic per-key version** — record `sentVersion` at send; on success clear the key **only if**
`version === sentVersion`; on failure never clear. This is the bug every naive implementation ships.

**H2** falls out of H1: the flush reads the value from the map **at send time**, never a captured
variable. (This is precisely what TanStack's `useMutation` retry structurally cannot do — it
re-sends captured variables and can mark a cell saved with a stale value.)

**H5 — reentrancy.** A key already in flight must be skipped this tick even if dirty, or two
requests for one cell race and the older can land last.

**H3** — no unbounded growth: LWW *replaces*, so the pending map is bounded by |keys|. Cap backoff
at 30s; keep only the latest error per key. A key removed from the source while dirty is only ever
dropped via explicit `discard(key)` — a write is never silently lost.

**H4** — within a tick, no ordering: parallel `allSettled`. Safe **only** because key independence
is a stated precondition. Document it as a precondition, not an assumption.

**H6** — never start the timer during SSR; stop on `onScopeDispose`; the timer must not run while
the outbox is clean.

## Smart defaults (standards: the bare form must be right with no options)

```ts
const cells = reactive<Record<string, string>>({ A1: 'foo' })
const outbox = useWriteBehind(cells, (value, key) => api.put(`/cell/${key}`, value))
cells.A1 = 'bar'   // that is the whole API
```

interval **1000ms**, timer only runs while dirty · `Promise.allSettled` over dirty keys · response
**ignored**, no opt-in · retry **forever**, per-key backoff 1→2→4→8→16→30s capped, always re-sending
the **current** value (silently dropping a user's edit is the one unacceptable outcome) · unlimited
batch · flush on `visibilitychange → hidden` (not `beforeunload`, unreliable on mobile), best-effort
and documented as such · stops on scope dispose, never starts on the server.

Batch endpoint is the one other shape worth first-class support:
```ts
useWriteBehind(cells, { flush: entries => api.patch('/cells', Object.fromEntries(entries)) })
// throw → whole batch stays pending;  return { failed: [keys] } → partial
```

Returned object is reactive and enriched (house style — do not make consumers assemble state):
`pending` / `inFlight` / `failed` / `isSyncing`, plus `set(key, value)`, `flush()`, `retry(key?)`,
`discard(key)` — `discard` being the only way to lose a write.

Every option is opt-**out** shaped: `interval`, `debounce`, `retry`, `flushOnHidden`, `keys`, `equals`.

## Refuse list — put it in the README

No IndexedDB/persistence (README recipe hooking `pending` instead) · no offline detection
(*offline is not a special case, it is a failing flush*) · no conflict resolution or merge (needs a
CRDT) · **no reading at all** — it is write-only · no ordered operation log or cross-key
transactions · no HTTP client, transport or `sendBeacon` · no schema or collections.
Each of these is a step toward RxDB/Replicache/TanStack DB, where a solo package loses on day one.

## Surface

**Composable is the product.** The recorded directive-first preference exists for DOM-behaviour
libraries; this one's product is a store. An optional `v-write-behind="'A1'"` directive is a
*second* surface, built after — not in this ticket.

## Acceptance

- TDD. The core is a state machine plus a clock: `vi.useFakeTimers()` and a fake flush of
  controllable deferred promises. Must pin: N edits → 1 call · edit-during-flight → exactly 2 calls,
  the second carrying the newest value · success does **not** clear a key whose version advanced ·
  rejection keeps the key pending · retry sends the *current* value, not the sent one · the backoff
  schedule · one rejection does not block sibling keys · never two in-flight requests for one key ·
  timer stops when clean · dispose cancels.
- **Mutation-test the suite** — two agents did this today and both found their tests were weaker
  than they looked. At minimum: replace the version guard with a boolean and confirm the
  edit-during-flight test goes red; remove the reentrancy skip and confirm a test goes red.
- Vue-first, typed, no `@ts-ignore`, no casts. `package.json` per the repo template; `npm pack
  --dry-run` shows no source/test leakage. `ARCHITECTURE.md` with the module map and the invariant.
- Name `vue-write-behind` (verified free). Do not publish.

**What phase 1 cannot prove, and must be recorded as UNPROVEN, not asserted:** `visibilitychange`
actually firing and the request actually leaving (`keepalive`), real offline, and — the one that
matters most — **that the cell does not jump when a slow server responds while the user is still
typing in it.** That is the whole product and no unit test can demonstrate it. It is WBC-3's card.
