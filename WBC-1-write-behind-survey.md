# WBC-1 — write-behind cache: uniqueness survey, then scope

**Read `tickets/_STANDARDS.md` first.** Read-only research; produce a verdict, change nothing.
Same cancellation-enforced uniqueness bar as KB-1 — "do not build this" is a valid verdict.

## The owner's description, verbatim
> "a key based local cache which will keep latest states of a data set and update in intervals,
> assume like this. I have an excel table, which user will update meanwhile and in the background
> we will update server with it's latest state, so we will do Promise.allSettled(cache) and only
> if succesfully updated update our cache. ... Vueuse has usePromiseQueue or something, but this
> isn't it, it should watch based on a new key entry AND package their changes to update stuff in
> the background, without returning the result because that will cause a jump."

## The pattern, named
A **write-behind (write-back) cache with per-key coalescing** — the client-side outbox pattern.
Four distinguishing properties; test every incumbent against all four:
1. **Per-key last-write-wins coalescing.** A second edit to a key *replaces* the pending one
   rather than queueing behind it. N edits to one cell -> one request.
2. **The server response is deliberately not applied to local state.** Local is authoritative
   for the user; echoing the server value back is the "jump" — it clobbers the cell still being
   edited. This is the *opposite* of what most cache libraries do on mutation success.
3. **Interval/batched flush via `Promise.allSettled`** — one failed key does not block others.
4. **Failure keeps the key pending**, so the next flush retries it carrying whatever newer value
   the user has typed since.

## Survey
VueUse (the owner names `usePromiseQueue`, which I do not believe exists — check what actually
does: `useAsyncQueue`, `useDebounceFn`, `useThrottleFn`, `useMemoize`, `watchDebounced`,
`useStorage`/`useStorageAsync`). **TanStack Query / `@tanstack/vue-query` is the serious
incumbent** — `useMutation`, `onMutate` optimistic updates, MutationCache, `scope`,
`networkMode`, `persistQueryClient`, offline replay. Answer directly: **can TanStack express
per-key last-write-wins coalescing and response-suppression today, and in how much consumer
code?** If the answer is "30 lines", the package probably should not exist — say so.
Then: Pinia persistence plugins, swrv, vue-request, villus/urql offline exchanges, Apollo
optimistic UI; and the local-first tier — RxDB, Dexie (+dexie-cloud), PouchDB, Replicache,
ElectricSQL, Yjs/Automerge. npm searches: `write-behind`, `write-back cache`, `outbox`,
`offline queue`, `mutation queue`, `debounced sync`, `autosave`, `sync-engine`.

## Answer
1. **Is there a wedge?** Test it hardest against TanStack Query.
2. **Framework:** **Vue-first — the owner has decided this** (`_STANDARDS.md`). Do **not**
   return "ship it as framework-agnostic TypeScript with a thin Vue adapter". Design the Vue
   API; note internally testable pure core if that falls out, but the product is Vue-first.
3. **Scope boundary — most likely to sink this.** Where does it stop? Persistence across
   reloads (IndexedDB), offline detection, conflict resolution, retry/backoff, N requests vs
   one batched call, `visibilitychange`/`beforeunload` flush via `sendBeacon`. Each addition
   moves it toward RxDB/Replicache, where it cannot win. Recommend the smallest genuinely
   useful version and name what it must **refuse** to do.
4. **The hard problems** a naive version gets wrong: a key edited *while its own flush is in
   flight* (the response must not clear a value newer than the one sent — needs per-key
   versioning, not a boolean); reconciling a failed key whose value has since changed;
   unbounded growth of a permanently-failing key; whether cross-key flush order can matter.
5. **DX and smart defaults** (`_STANDARDS.md`): what does the bare form do with no config?
   What interval, what retry policy, what batch size — chosen so the common case needs nothing?
6. **Name.** Check npm availability and recommend one matching the wedge.
7. **Testability.** Pure logic, no DOM: fast deterministic unit tests with fake timers. Say what
   still needs a real browser (persistence, `sendBeacon`, `visibilitychange`, actual offline) —
   those become playground cards and browser checks, per `_STANDARDS.md` item 4 and 5.

**Return:** verdict up front; incumbent table with real numbers; the wedge as a README lead;
recommended scope with an explicit "will not do" list; the hard problems; the name; and only if
"build", a Vue-first API sketch. Cite what you checked; assert no download count from memory.
