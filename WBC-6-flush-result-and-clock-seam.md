# WBC-6 — `flush()` cannot report what it did, and the composable has no clock seam

Two API-shaped follow-ups from the 0.1.1 audit fix. Both were deferred because they change or
add public surface, and 0.1.1 was a patch release for a data-loss bug — the fix had to be
reviewable on its own. Neither is a correctness defect today.

## 1. `flush(): Promise<void>` resolves identically whether it worked or not

Audit finding 4, second half. 0.1.1 fixed the behaviour — `flush()` now sends keys parked by
`retry: false`, which it silently skipped before, so the tab-hidden last-chance save is no
longer inert — and documented that resolving is not proof of success: the caller is told to
read `pending` / `failed` afterwards.

That is honest but awkward at the one call site that matters:

```ts
window.addEventListener('pagehide', async () => {
  await outbox.flush()
  // …did anything land? Only `outbox.failed` and `outbox.pending` can say.
})
```

**Proposal (0.2.0):** resolve with a summary — `{ sent, settled, failed }` as key arrays, or a
`boolean` for "everything I started landed". Widening the resolved value from `void` is
technically source-compatible for callers that ignore it, but it is public API, so it goes in a
minor. Decide the shape against the two real uses: a save button and an unload handler.

## 2. Two of the three time sources still bypass the injected clock

Audit finding 13. `flush.ts` takes `now: () => number` and `outbox.ts` takes every clock
reading as an argument (including `failures(now)`, added in 0.1.1). `useWriteBehind.ts` now owns
exactly one `Date.now()` call site and hands the reading down, and `ARCHITECTURE.md` says so
instead of claiming the whole split is clock-free — but the composable still cannot be driven by
a fake clock from the outside, so every Vue-level test needs `vi.useFakeTimers()` plus
`vi.setSystemTime`.

**Proposal:** an undocumented-or-documented `now?: () => number` option (and possibly a timer
factory) so the composable can be tested deterministically through the seam the architecture
advertises. Weigh it against the DX rule in `tickets/_STANDARDS.md`: an option that exists only
for tests is a smell, so either it earns its place (custom clocks are a real use case in
offline-first apps) or the architecture doc should keep saying plainly that the composable owns
the clock.

Note the fake-timer suite is not weak for lacking this: the 0.1.1 run mutation-tested it at
58/59 killed, including a test that moves the system clock **backwards** mid-wait to prove a
queued write is not stranded by a clock adjustment.
