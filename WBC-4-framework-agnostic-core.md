# WBC-4 — extract a framework-agnostic `@ozjsey/write-behind`, and make the Vue package consume it

**Owner request, 2026-09-14:** *"Write behind doesn't really need to be Vue, make a typescript
version of it as well, and most ideally vue package uses the package."* Followed by: *"Net new
project for sure tho."*

So: a **new top-level package**, not a refactor in place — and `vue-write-behind` becomes a thin
adapter over it rather than a fork of it. Two packages, one engine, no duplicated state machine.

---

## Why this is mostly already done

Read `vue-write-behind/ARCHITECTURE.md` first. The split it describes is already the right one; it
just stops one layer short of a package boundary. Measured coupling to Vue across `src/`:

| Module | Vue? | Destination |
|---|---|---|
| `outbox.ts` | none — no Vue, no timers, no I/O | core, unchanged |
| `scheduler.ts` | none | core, unchanged |
| `flush.ts` | none | core, unchanged |
| `types.ts` | one erased `import type { Ref } from 'vue'` | core, with `Ref` removed |
| `lifecycle.ts` | `isServer` / `onTabHidden` are DOM-only; **only `onDispose`** uses Vue | **split** |
| `useWriteBehind.ts` | `isRef`, `shallowReactive`, `watch` | **split** |

Three of six modules move untouched. That is the whole reason this is worth doing.

## What the core package is

- Folder: `write-behind/` at the repo root. Name: **`@ozjsey/write-behind`** (verified free on the
  registry 2026-09-14, as is the unscoped `write-behind`). Version **0.1.0**.
- **Zero dependencies. No `peerDependencies`.** Node and the browser both.
- Same conventions as every other package here: `src/`-split single-purpose modules behind a thin
  re-export entry (`writeBehind.ts`), its own `ARCHITECTURE.md` naming the module map and the
  invariant, `CHANGELOG.md`, `README.md`, tsup build to `dist/*.min.js` + `.cjs` + `.d.ts`.

### The API

```ts
import { createWriteBehind } from '@ozjsey/write-behind'

const cells: Record<string, string> = { A1: 'foo' }

const wb = createWriteBehind(cells, (value, key) => api.put(`/cell/${key}`, value))

cells.A1 = 'bar'
wb.sync()            // "I changed the record" — diff it and queue what moved
// or, equivalently, skipping the diff:
wb.set('A1', 'bar')  // writes through to the record AND queues
```

The engine cannot observe a plain object, so **`sync()` is the seam that `watch` fills in Vue.**
That is the only conceptual difference between the two packages.

Surface:

| Member | Contract |
|---|---|
| `set(key, value)` | write through to the source record, then queue. Unconditional — the escape hatch for a change `equals` cannot see and for keys `keys` filters out. Same semantics as today's `store.set`. |
| `sync()` | diff the source against the shadow map and queue what changed. Idempotent; cheap when nothing moved. |
| `flush()` | `sync()` then dispatch immediately. Returns the same promise shape as today. |
| `retry(key)` / `discard(key)` | unchanged from today. |
| `pending` / `inFlight` / `failed` / `isSyncing` | **snapshot getters**, not live arrays. Reading returns the current value; identity is stable while nothing transitions (reuse the existing `sameKeys` / `sameFailures` comparison so a subscriber can cheaply skip a no-op). |
| `subscribe(fn): () => void` | called after every outbox transition. This is what the Vue layer mirrors into `shallowReactive`. Returns unsubscribe. |
| `dispose()` | stop the scheduler, drop the visibility listener. Must be idempotent. |

`source` accepts `Record<Key, T>` **or** `() => Record<Key, T>`. The getter form is what the Vue
adapter passes (`() => isRef(source) ? source.value : source`); do not special-case Vue in the core.

Options are today's `WriteBehindOptions` minus nothing: `write` / `flush`, `interval`, `debounce`,
`equals`, `keys`, `retry`, `flushOnHidden`. `WriteBehindSource<T>` drops its `Ref` arm — that arm
moves to the Vue package's own source type.

### What moves out of `useWriteBehind.ts`

Everything except the three Vue calls: the shadow map and its seeding, `syncFromSource`'s diff, the
`tracked` key filter, `equals`, the `now()` / `readyAt()` clock, `reschedule()`, `onOutboxChange`,
and the `sameKeys` / `sameFailures` comparators. Put it in `src/core.ts` (or split further if it
reads long — use judgement, the rule here is one purpose per module).

`lifecycle.ts` splits: `isServer` and `onTabHidden` go to the core untouched; `onDispose` stays in
the Vue package, as the only thing that needed Vue.

## What the Vue package becomes

`useWriteBehind`'s **public surface must not change by one character.** It is published at 0.1.1 and
people are using it. Same signature, same returned store shape, same type names, same docs. This is
a re-plumbing behind a stable API, released as **0.2.0**.

```ts
export function useWriteBehind<T>(source, writerOrOptions): WriteBehind<T> {
  const core = createWriteBehind<T>(() => (isRef(source) ? source.value : source), writerOrOptions)
  const store = shallowReactive<MutableWriteBehind<T>>({ /* mirrors core, delegates methods */ })
  core.subscribe(() => { /* copy snapshots in, identity-stable */ })
  watch(() => (isRef(source) ? source.value : source), () => core.sync(), { deep: true })
  onDispose(() => core.dispose())
  return store
}
```

`store.flush()` must keep today's behaviour of reading the source *before* dispatching — the source
watcher is a `pre` watcher, so an edit made in the current tick has not been seen yet, and a save
button next to an input would otherwise miss the last keystroke. `core.flush()` already does the
`sync()` first, so this falls out — **but there is a test for it, and it must still pass.**

### Dependency direction

`vue-write-behind/package.json` gains `"dependencies": { "@ozjsey/write-behind": "^0.1.0" }`.

**Publish ordering matters and is the owner's call:** the core must be on the registry before the
Vue 0.2.0 tarball can resolve. Do not publish anything. State the ordering plainly in the summary.

For local work, wire it the way this repo already resolves sibling packages — check how the
playground's `vite.config.ts` aliases `@ozjsey/*` specifiers and follow it. The build must treat
`@ozjsey/write-behind` as **external** (do not inline it into the Vue dist; that would ship two
copies of the state machine to anyone who installs both).

## Tests

These three files import **no Vue whatsoever** — verified — so they move to the core package as-is,
along with `test-utils.ts`:

| File | Declarations |
|---|---|
| `outbox.test.ts` | 50 |
| `scheduler.test.ts` | 15 |
| `flush.test.ts` | 16 |

The core's vitest config is plain — **no `vue-3.5` / `vue-3.3` projects**, because there is no Vue
to be compatible with. Keep a node-environment project so "imports cleanly with no DOM" stays
proven.

`vue-write-behind` keeps `useWriteBehind.test.ts` (51) and `vueWriteBehind.ssr.test.ts` (3), still
across Vue 3.5 / 3.3 / SSR. **Every one of those 54 must still pass unchanged** — that is the
evidence the surface did not move. If you find yourself editing an assertion in
`useWriteBehind.test.ts`, stop: either the refactor changed behaviour it should not have, or you
have found a real bug that deserves saying out loud rather than editing around.

**New tests are owed in the core**, because the logic extracted from `useWriteBehind.ts` currently
has coverage *only* through the Vue composable. At minimum: the shadow seeding (a record's initial
contents are seeded, **not** queued — whatever it started with came from the server), the `equals`
short-circuit, the `keys` filter in both array and predicate form, a key deleted from the source
keeping its queued write, `set()` on a filtered-out key still queueing, `sync()` being idempotent,
`subscribe` firing on transitions and its unsubscribe working, and `dispose()` being idempotent and
actually stopping the timer.

## Acceptance

- [ ] `write-behind/`: `npm test` green, `npm run build`, `npm pack --dry-run` shows no source or
      test leakage, `npm run check:dist` (port the existing `dist-check.mjs` pattern) proves the
      built artifact behaves — this repo has shipped a stale `dist/` three times.
- [ ] `vue-write-behind/`: all 54 existing declarations pass **unedited**, across all three vitest
      projects. Build clean. Ships no second copy of the engine.
- [ ] Both packages have `README.md`, `ARCHITECTURE.md` (module map + the invariant), `CHANGELOG.md`.
      The core README must stand alone for someone who has never heard of Vue, and lead with the
      thing that makes it worth installing — read `vue-write-behind/README.md` and
      `ARCHITECTURE.md` for what that is: **the version guard that stops a slow response from
      clearing a newer edit.** Every naive write-behind ships that bug.
- [ ] `vue-write-behind`'s README and ARCHITECTURE say it is now a layer over the core, and its
      copy-paste note is corrected — lifting `src/` alone no longer works, but lifting the core
      package's `src/` does.
- [ ] The core's `ARCHITECTURE.md` keeps the "Nothing outside `outbox.ts` may clear a dirty key"
      invariant as its headline. It is the reason the split is shaped this way.

## Out of scope — a separate ticket (WBC-5), do not start it

Playground tab, documentation view, interactions spec, standalone `playground.html`, README
deep-links. That lands after this does.

## House rules that apply

`instructions/CONVENTIONS.md` and `tickets/_STANDARDS.md` are binding. TypeScript-first, exported
types, no `@ts-ignore`, no defensive branches for states the types exclude, simple stupid code.
**Source is the distribution** — people copy these files rather than installing them, so readability
of `src/` *is* the product.

**Do not publish anything.** Do not run `npm publish`. The owner publishes.
