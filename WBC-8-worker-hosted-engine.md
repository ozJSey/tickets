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

## DECIDED 2026-09-20 (FINAL) — `SharedWorker`. Owner: *"You can do SharedWebworker than it's fine."*

Settled after measuring the thing both earlier answers were guessing at. Stable-URL workers, real
Chrome: a dedicated `Worker` never survives a refresh (`W856…` → `W184…`, and again `W622…`); a
`SharedWorker` survives one **only while another tab holds a port** (`S889392295` before and after),
and dies with the last port when it is the only tab.

That kills the "persists between routes/refreshes" rationale for the dedicated tier outright, and
leaves SharedWorker as the only variant buying anything beyond off-main-thread CPU and fault
isolation. Durability past a close is neither — see WBC-9, which is the right home for it.

The planner's earlier "buys nothing" was still wrong for the reason recorded below (the writer is
consumer code and can be arbitrarily expensive), and the owner's restriction concern is still real:
no SharedWorker on Chrome for Android, so **the fallback is the common path on mobile and must be
explicit and reported, never silent.**

### Superseded reasoning, kept so it is not re-argued


Owner: *"I wanna say one each, that is the expected behavior when we modify WebWorker like that it
may not work, WebWorker is pretty restricted."*

This reverses the planner's first answer, and the reversal is recorded because the first answer was
wrong on a point of fact, not taste.

**The planner argued a dedicated worker "buys nothing" because the engine's work is a timer, a Map,
an equality check and a fetch — none of it CPU-bound. That analysis omitted the writer.** The
writer is CONSUMER code, and it is the one part of this library that can be arbitrarily expensive:
serialising a large document, signing a request, compressing a payload, `JSON.stringify` over
something big. All of that runs inside the writer, and a dedicated worker moves it off the main
thread. The tier is justified on its own, with no cross-tab argument needed.

**And the restriction argument points the other way from how the planner weighed it.**
`SharedWorker` is the MORE restricted of the two: absent entirely on Chrome for Android, harder to
debug, and its port lifecycle — last-tab detection, pruning a port whose tab died without
`pagehide` — is a problem that exists ONLY because of the sharing. One worker per instance deletes
that entire class of failure. Cross-tab dedupe is a real property, but it is not worth buying with
the least portable API in the pair.

**"It may not work" is the expected behaviour, not a defect to engineer around.** A worker module
that does not provide all context throws at load, naming the missing symbol — the runtime enforces
the rule so the docs do not have to. What the library owes is to report that honestly and stop,
which is now cheap: engine 0.1.2 gives a failing key five retries and then blocks it, so a worker
that can never load fails in about half a minute and says so, instead of retrying every 30 seconds
for the life of the page. Before 0.1.2 this ticket would have shipped an infinite loop.

What worker mode may therefore claim:

- moves the consumer's writer off the main thread — the only performance claim, and it is the
  consumer's payload that makes it true or trivial
- opt-in forever; the consumer authors the module, because a closure cannot cross `postMessage`
- explicit, reported fallback when `Worker` is unavailable or the module fails to load — never a
  silent downgrade
- **no** cross-tab single queue, **no** surviving the tab that issued the write, **no** durability
  past a page close. Each tab keeps its own queue and its worker dies with it. If surviving a close
  is the goal, that is a persisted outbox and a different ticket — `snapshot.ts` is a view-diffing
  layer for UI subscribers, not storage.

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

### D. Writers become a context-neutral module — enforced by the runtime, not by prose

The consumer's writers must be importable from both the worker entry (A) and the explicit fallback
(B). The README shows the shape once: `writers.ts` imports nothing DOM-flavoured.

**Owner, 2026-09-17:** *"When they give all context WebWorker part isn't hard too, WebWorker already
warns what is not defined in their scope."* — and that is the point that de-risks this whole
ticket. A worker module referencing `document`, `window`, or a closed-over variable **throws at
load, in worker scope, naming the symbol.** The "provide all context" rule is not a convention the
docs have to carry and consumers have to remember; it is enforced by the platform, loudly, on the
first run. That is a *better* failure mode than the main-thread path, where a badly-built writer
fails later and quieter.

So do not write defensive validation to re-check what the runtime already rejects, and do not
apologise for the constraint in the README — state it once, show the two-file shape, and let the
error message do the teaching. The negative-control test demonstrates the failure so the message
is on record.

**What is genuinely fiddly here is NOT this** — it is port lifecycle (§C): knowing which tab is the
last one, and pruning a port whose tab died without firing `pagehide`. Spend the care there.

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

## Publish sequencing — TWO packages move, and the second one is not optional

**Owner, 2026-09-17:** *"I would accept write behind v 0.2.0 and v write behind 0.2.1 or something."*

| package | from → to | why |
|---|---|---|
| `@ozjsey/write-behind` | 0.1.0 → **0.2.0** | a worker host is a real feature; minor |
| `@ozjsey/vue-write-behind` | 0.2.0 → **0.2.1** | patch whose ONLY job is `^0.1.0` → `^0.2.0` |

**The adapter bump is load-bearing, not bookkeeping.** On `0.x` versions a caret pins the minor:
`^0.1.0` means `>=0.1.0 <0.2.0`, measured —

```
engine 0.1.1  satisfies "^0.1.0"?  YES
engine 0.2.0  satisfies "^0.1.0"?  NO
```

So if the engine ships 0.2.0 and the adapter's range is left alone, **every Vue consumer silently
keeps resolving the 0.1.x engine.** No error, no warning — the worker code is on npm and simply
never reaches them. That is the worst failure shape this portfolio has: a feature that looks
shipped and is not.

The adapter still gains **no worker code** (owner: *"I wanna maintain one"*). Its 0.2.1 is the
dependency range, its CHANGELOG entry, and nothing else.

Order is already correct in `scripts/publish.mjs`'s hard-coded `ORDER` — engine before adapter —
so both flip to PUBLISH in one run. See **PUB-6** for the gate that makes this class of mistake
impossible to ship rather than something the next person has to remember.
