# WBC-9 — a persisted outbox: survive refresh, navigation and offline

**Owner, 2026-09-20:** *"We can add a config to survive route change page refresh offline etc.. It
doesn't need to be webworker doing that, but we need to do that."*

Correct on both counts, and the measurement backs it: **no worker gives you this.** Driven in real
Chrome against stable worker URLs —

| scenario | dedicated `Worker` | `SharedWorker` |
|---|---|---|
| refresh, only tab open | dies (`W856…` → `W184…`) | dies (`S386…` → `S889…`) |
| refresh, another tab holding a port | dies (`W622…`) | **survives** (`S889392295` both sides) |

A dedicated worker never survives a refresh. A SharedWorker survives one only while another tab
keeps a port open, and dies with the last port — so neither covers "close the tab and come back",
and neither covers offline at all. Durability is a STORAGE problem, not a threading one. WBC-8 may
proceed on its own merits; it is not this.

## What it has to do

1. **Persist the outbox** — pending values, their keys, `attempts`, and the failure state — on
   every transition, and rehydrate on load.
2. **Replay on load**, respecting `maxRetries` (engine 0.1.2): a key that was blocked stays blocked
   until a fresh edit or an explicit `retry()`, or it will burn its ceiling again on every reload.
3. **Offline** — `navigator.onLine` plus the `online` event as a wake source, so a queue that
   failed while disconnected flushes on reconnect instead of waiting out a backoff.

## Default ON — and why that is not a reversal of VC-1

**Owner, 2026-09-20:** *"I wanna say that is the expected behavior so it should be default true."*

This looks like it contradicts the VC-1 precedent quoted below, and it does not. VC-1 was about
v-copy's clipboard HISTORY: arbitrary user content, retained indefinitely, powering a convenience.
Writing that to someone's disk unasked is exactly the cluttering the owner objected to.

The outbox is the opposite case on every axis that mattered there:

- it holds **unsaved work in flight**, not a record of past activity
- every entry is **deleted the moment it lands** — the steady state is empty
- **losing it is the failure this library exists to prevent.** Default-off means the central
  promise silently stops holding across a refresh, which is the one thing the README may not do.

So: persistence is ON by default, with a real store, and `persist: false` opts out.

### What default-on obliges us to get right

- **A default store must be chosen, not improvised.** IndexedDB: async, quota far above
  localStorage, and it does not block the main thread on a large draft. localStorage is the
  fallback only if IndexedDB is unavailable, and an explicit adapter overrides both.
- **Staleness has to be bounded.** Replaying a write the user abandoned three days ago is its own
  bug. Entries carry a timestamp and a `maxAge` (default measured in hours, not days); anything
  older is dropped on rehydrate and REPORTED, never silently.
- **Opting out must be one word.** `persist: false`, and it must also be reachable per-key for the
  consumer who has one field they refuse to put on disk.
- **The README must say plainly that unsaved values are written to disk by default**, and how to
  turn it off. A default that surprises a security reviewer is a default that gets ripped out.

## The consumer can still choose the storage.

Owner precedent, from VC-1: *"I don't wanna clutter local storage without asked… we need them to
choose the way to store it."* So the option takes an adapter — `{ read, write }`, sync or async —
and ships NO default storage. Omitted means no persistence, exactly as today.

That also sidesteps the parts a built-in default gets wrong: quota, serialisation of a generic `T`,
multi-tab writes to one key, and whether anything sensitive is being written to disk at all. Those
are the consumer's questions and only they can answer them.

## Landmines

- **`T` is generic.** The engine cannot assume JSON. Serialisation belongs to the adapter.
- **Two tabs, one store.** Rehydrating in tab B a write that tab A already has in flight will
  double-send it. Needs a claim/lease or a last-writer-wins rule, decided before any code.
- **A poisoned entry must not brick startup.** If rehydration throws on one key, drop that key,
  report it, and continue — never fail the whole load.
- **This belongs in `@ozjsey/write-behind`, not the Vue package** — WBC-7's rule: *"That's kind of
  why I want 2 packages I wanna maintain one."*
