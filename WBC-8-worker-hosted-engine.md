# WBC-8 — worker-hosted engine: opt-in, consumer-provided context

**Owner, 2026-09-16:** *"For writeBehind, we need to be introducing WebWorker use conditionally.
Because WebWorker will not use the main thread and it will persist a lot more cleanly between
different routes in web apps."* — then, closing the design's central question himself:
*"Webworker is optional because you need to provide all context, so it cannot be default."*

That second sentence is the constitution of this ticket. Writers are consumer **closures** — auth
headers, serializers, endpoints — and closures do not cross a worker boundary. So the library can
never conjure a worker itself: **the consumer authors the worker module that provides all context,
and worker mode is opt-in forever.** Anything that tries to make it a default is doing it wrong.

**All of it lands in `@ozjsey/write-behind`, none of it in the Vue package** — same rule as WBC-7
(*"That's kind of why I want 2 packages I wanna maintain one"*). The Vue adapter must work
unchanged against a worker-backed engine, which makes task E an interface question, not a feature.

## What worker mode actually buys — claim only what is true

| Context | Dedicated `Worker` | `SharedWorker` | Claim in README |
|---|---|---|---|
| SPA route changes | main-thread relief only — module state already survives SPA routing | same, plus below | **Never** claim "survives routes" for SPAs; the singleton already does |
| MPA / hard navigations | dies with the page — **no** persistence win | queue survives while any same-origin context lives | This is the owner's "persist between routes" case — SharedWorker only |
| Multi-tab | one queue **per tab** (status quo) | **one queue per origin** — same-key writes from two tabs coalesce instead of racing | The boldest consequence; nothing on npm ships it |
| Heavy payloads | serialization + retry timers off the main thread | same | True for both |

The honest headline is the SharedWorker one. A dedicated `Worker` is the degraded sibling
(Android WebView has no SharedWorker), not the lead.

## Design

### A. `@ozjsey/write-behind/worker` — the host half

New export path (own entry in `exports`, so main-thread bundles never pay for it):

```ts
// wb.worker.ts — the CONSUMER writes this file; it IS the "provide all context" step
import { hostWriteBehind } from '@ozjsey/write-behind/worker'
import { writers } from './writers'          // context-neutral module, see D

hostWriteBehind({ writers, retry: { attempts: 5 } })
```

`hostWriteBehind` detects its global (`SharedWorkerGlobalScope` → `onconnect`, ports tracked;
`DedicatedWorkerGlobalScope` → single implicit port) and runs the **existing engine** behind a
message protocol. No fork of the engine: it is constructed exactly as on the main thread.

### B. `connectWriteBehind` — the client half

```ts
const wb = connectWriteBehind(
  () => new SharedWorker(new URL('./wb.worker.ts', import.meta.url), { name: 'write-behind' }),
)
wb.write('doc:42', patch)   // same public surface as createWriteBehind — pinned, see Acceptance
```

- Returns the **same public interface** as `createWriteBehind`. Parity is enforced by a type-level
  test (`Expect<Equals<WorkerClient, Engine>>`-style), not by promise.
- State (`pending`, `inflight`, counts) mirrors via snapshot-then-patch messages; reads are
  synchronous against the mirror, never round-trips.
- **No silent fallback.** If the factory throws (CSP `worker-src`, SSR, no `SharedWorker`),
  `connectWriteBehind` throws unless the consumer passed `fallback: () => createWriteBehind({ writers })`
  — explicitly, with their own main-thread writers. A silent fallback would mean two code paths
  in production and no one knowing which ran; that is the PG-21 lesson wearing a new hat.

### C. Lifecycle over the boundary

The worker global has no `document`, so WBC-7's `visibilitychange`/`pagehide` pair cannot live in
the worker. The split:

- **Client** keeps the WBC-7 listeners and forwards them: `port.postMessage({ t: 'lifecycle', reason: 'pagehide' })`.
  `postMessage` enqueues synchronously during `pagehide`; delivery is guaranteed while the worker
  outlives the page — which is the SharedWorker case by definition when any other tab is open.
- **Worker** treats a forwarded `pagehide` from its **last connected port** as what it is: the
  origin going away. It flushes with `{ reason: 'unload', final: true }` writer context (WBC-7's
  contract, unchanged) so writers can set `keepalive` — `fetch` keepalive works in worker scope.
- Port disconnect detection: forwarded `pagehide` is primary; a heartbeat (client pings every 15s,
  worker prunes silent ports at 45s) is the backstop for crashed tabs that never fired it.
  **Do not** rely on MessagePort `close` events — availability is too new to carry the guarantee.

### D. Writers become a context-neutral module — documented, not enforced

The consumer's writers must be importable from both the worker entry (A) and the explicit fallback
(B). The README shows the shape once: `writers.ts` imports nothing DOM-flavoured. The library
cannot enforce this; the negative-control test (below) demonstrates the failure mode instead.

Engine-side prerequisite, verify before building: `src/` must reference no DOM global outside the
lifecycle module. If WBC-7 left `document`/`window` reachable from engine construction, moving it
behind an environment seam is part of this ticket, not a follow-up.

### E. Protocol + serialization

- Versioned first message (`{ t: 'hello', v: 1 }`); mismatch throws loudly on connect.
- Write payloads cross via structured clone. A non-cloneable payload **throws synchronously at
  `write()` on the client** — strictly better DX than main-thread mode, where a closure smuggled
  into a payload fails later and quieter. Say so in the README.
- Rejected alternative, recorded: `navigator.locks`-based cross-tab coalescing without any worker.
  Lighter, but it serializes flushes rather than unifying the queue, cannot take work off the main
  thread, and adds a second concurrency model. One mechanism, not two.

### F. Vue package: zero new code, one seam check

`@ozjsey/vue-write-behind` must accept any object satisfying the engine's public interface. If the
adapter currently calls `createWriteBehind` itself rather than accepting an engine, add the
injection point — that is the entire Vue-side change. Worker mode in a Vue app is then:
`app.use(VueWriteBehind, { engine: connectWriteBehind(...) })`.

## Non-goals

- **No Service Worker mode.** Background-sync-style replay after the last tab closes is workbox's
  niche and a different lifecycle model; out of scope, note it in the README's honesty table.
- **No bundler plugin** to generate the worker entry. `new URL(..., import.meta.url)` is the
  platform way and every current bundler understands it.
- **No default-on anything.** See the first paragraph.

## Acceptance — standing criteria apply (BOARD.md), plus:

1. Type-level parity test between client and engine public surfaces; a drift is a red build.
2. Protocol unit tests over a mock `MessagePort` pair (jsdom-safe): snapshot/patch mirroring,
   lifecycle forwarding, heartbeat pruning, version mismatch, non-cloneable payload throw.
3. **Real-browser check (playground):** a card running an actual SharedWorker with two iframes
   sharing one queue — same-key writes from both coalesce to one flush. Negative control: the same
   card against two main-thread engines shows the double flush the feature deletes.
4. **Negative control for B:** factory that throws + no `fallback` → loud error, nothing enqueued
   anywhere; with `fallback` → main-thread engine runs and says so via a `mode` field.
5. Last-tab keepalive flush: **UNPROVEN in CI is acceptable** — automating "close the final tab
   and observe the server" is not honestly reachable from the harness; mark it UNPROVEN in the
   README per the standing rule, with a manual verification recipe.
6. README gains the "what worker mode buys" table from this ticket, verbatim in substance —
   including the SPA row that says the singleton already survives SPA routing. No overclaim.

## Publish sequencing

The engine's 0.1.0 is live on the registry (published 2026-09-16). This lands as the **next
patch-level bump of whatever is current** — owner rule, 2026-09-16: *"We version too generously,
stop it"* — no dedicated minor for it; additive export path, no breaking change to the main entry,
and the Vue adapter's dependency stays `^0`-ranged until 1.0.
