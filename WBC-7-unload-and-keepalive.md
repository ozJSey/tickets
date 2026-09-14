# WBC-7 — surviving a page refresh: `pagehide`, and `keepalive` on the request

**Owner, 2026-09-14:** *"In writebehind, do we support onunload and keep alive?"* — then, clarifying:
*"What I meant was keep-alive actually JavaScript functionality to keep promises alive on page
refresh etc"*, and *"If not please add these and necessary tests."*

So this is about `fetch(url, { keepalive: true })` / `navigator.sendBeacon` — a request outliving the
page — **not** Vue's `<KeepAlive>` component.

**All of it lands in `@ozjsey/write-behind`, none of it in the Vue package.** Owner: *"That's kind of
why I want 2 packages I wanna maintain one."* The Vue adapter inherits every bit of this for free by
delegating, and must gain no unload logic of its own.

## Where it stands, measured

Probed in jsdom under both supported Vue versions (identical results; probe preserved at
`scratchpad/lifecycle-probe.test.ts`).

| Link in the chain | State | Evidence |
|---|---|---|
| The page going away is noticed | **partial** | `visibilitychange → hidden` flushes. **`pagehide` does nothing** — measured `[]` |
| The flush includes not-yet-due edits | **yes, already correct** | `dispatch(force)` ignores the debounce clock, the backoff and `retry: false` — so the last keystroke is included |
| The request survives the page | **no** | nothing tells the writer this is the final flush, so nothing can set `keepalive` |

The first and third are the two ways a write is lost on refresh, and the library currently has both.
That contradicts its own headline promise — *"the only operation here that loses a write is
`discard(key)`, and a consumer has to call it."*

---

## A. Listen for `pagehide` as well as `visibilitychange`

`src/visibility.ts` listens only to `visibilitychange`. Its own comment names the reason
`beforeunload` was rejected:

> *"mobile browsers routinely discard a page without ever firing `beforeunload`, and Safari fires
> `pagehide` instead."*

Rejecting `beforeunload` is right. Not then listening for `pagehide` is the incomplete half —
measured, a `pagehide` with no `visibilitychange` flushes **nothing**.

Platform guidance (Page Lifecycle API) is to take both: `visibilitychange` covers backgrounding,
`pagehide` covers same-tab navigation, refresh and tab close, and on iOS Safari the two do not
reliably coincide.

**Fix:** subscribe to both, **de-duplicated** so a browser firing both does not flush twice. One
place, ~12 lines. Keep rejecting `beforeunload`.

## B. Tell the writer when it is the final flush

This is the part the owner asked for, and the library cannot do it for you — the writer is a
consumer function, by deliberate design (*"You pass a function; what it does is your business"*).
What the library **can** do is say *when* the request has to outlive the page, so the writer can set
the flag.

Add a third argument to `write`, and a second to `flush`. Both are additive — every existing writer
ignores the new parameter and keeps compiling.

```ts
export interface WriteBehindAttempt {
  /** What asked for this flush. */
  reason: 'scheduled' | 'manual' | 'unload'
  /** True when the page is going away: the request must outlive it. */
  final: boolean
  /** 1 on the first try for this key, incrementing per retry. */
  attempt: number
}

export type WriteBehindWriter<T> =
  (value: T, key: WriteBehindKey, attempt: WriteBehindAttempt) => unknown

export type WriteBehindBatchWriter<T> =
  (entries: [WriteBehindKey, T][], attempt: WriteBehindAttempt) =>
    WriteBehindBatchOutcome | void | Promise<WriteBehindBatchOutcome | void>
```

The recipe this exists for:

```ts
createWriteBehind(cells, (value, key, { final }) =>
  fetch(`/cell/${key}`, {
    method: 'PUT',
    body: JSON.stringify(value),
    keepalive: final,          // survives the page going away
  }),
)
```

`attempt` is included because it is free here and consumers keep asking for it (idempotency keys,
"give up after N" logic in the writer). If it complicates the outbox, drop it and say so — `reason`
and `final` are the requirement.

### The constraints that must be documented, because they bite silently

- **`fetch` with `keepalive: true` is capped at 64 KiB** across *all* in-flight keepalive requests
  in the page. Over that, the request is rejected. A per-key writer firing 200 keepalive requests at
  unload will lose most of them — so **the batch writer (`flush`) is the better path for unload**,
  and the README should say so rather than leaving the reader to find out.
- **`navigator.sendBeacon` is POST-only** and shares that 64 KiB budget. It is the right tool when
  the endpoint accepts a POST; it cannot express a `PUT`.
- **Retries do not exist after unload.** The page is gone; there is no backoff to run. `final: true`
  means "this is the only chance", and a writer that would normally throw-and-retry should treat it
  differently. Say this plainly.

## C. Vue `<KeepAlive>` — a smaller, separate finding, already measured

Not what the owner meant, but it came out of the same probe and is worth keeping. Vue pauses a
deactivated component's effect scope, so the deep source watcher does not run:

```
edit while DEACTIVATED   → []                          nothing queued, nothing sent
then REACTIVATE          → ["A1=while-deactivated"]    the edit flushes on resume
then real unmount, edit  → (unchanged)                 disposal holds
```

**Deferred, not lost** — and arguably correct, since a cached invisible component should not be
firing requests. Identical on Vue 3.5 and 3.3, which is worth pinning: `EffectScope.pause()` is
3.5-only, so 3.3 reaches the same outcome by a different route.

It is **untested** (`useWriteBehind.test.ts` drives everything through `effectScope()` and never
mounts a component, so `<KeepAlive>` has never been exercised) and **undocumented**.

One property worth naming and pinning: the tab-hidden flush **does** rescue an edit made while
deactivated, because `flush()` calls `syncFromSource()` directly instead of waiting for the watcher —
the same decision the code justifies with *"a save button next to an input would miss the last
keystroke."* Measured: `["A1=edited-while-deactivated"]`.

This is the only item in this ticket that belongs to the Vue package.

## Acceptance

- [ ] `pagehide` + `visibilitychange`, de-duplicated, in `write-behind/src/visibility.ts`. Tested
      for either firing alone and both firing (exactly one flush).
- [ ] `WriteBehindAttempt` threaded to both writer forms. Tested: `reason`/`final` correct for a
      scheduled flush, a manual `flush()`, and an unload flush; existing 2-arg writers still work.
- [ ] README (core **and** Vue, since the Vue writer signature widens too): the `keepalive` recipe,
      a `sendBeacon` variant, the 64 KiB cap, why batching is better at unload, and that there are
      no retries after `final`.
- [ ] Vue `<KeepAlive>` suite that genuinely mounts components, run under both Vue projects, pinning
      all four behaviours above including the tab-hidden rescue.
- [ ] Both CHANGELOGs. **Check the registry before choosing a version** — `@ozjsey/write-behind`
      0.1.0 and `@ozjsey/vue-write-behind` 0.2.0 were both unpublished as of 2026-09-14, so this may
      fold into them for free. Do not assume; `npm view` first.

## Out of scope

`beforeunload` — correctly rejected, leave it rejected. Do not add an HTTP client or a built-in
transport; the writer stays the consumer's. A tiny exported `sendBeacon` *helper* is arguable if it
earns its place, but the `final` flag plus a documented recipe is the requirement, and a helper that
only wraps four lines probably does not earn it.
