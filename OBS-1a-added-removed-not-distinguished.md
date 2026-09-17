# OBS-1a — `children:added` and `children:removed` are not distinguished

**Owner, 2026-09-17: "Please fix it."** Carved out of OBS-1 so the correctness bug ships now,
without waiting on `each` (the L-effort delegated-observation feature). OBS-1 keeps `each`; this
ticket is the P1 only.

## The defect — confirmed by the planner, in the source

`src/mutate-records.ts:38`:

```ts
} else if (t === 'children:added' || t === 'children:removed') {
  childList = true
}
```

Both subscription types collapse into **one** `childList` flag. There is no record anywhere of
*which* of the two the consumer actually asked for, so a consumer subscribing to `children:added`
also receives every removal, and vice versa. The event names are a lie for half their users.

**Why 241 green tests never caught it:** not one test subscribes to one type and asserts the other
does *not* fire. The suite covers that both types deliver — never that they deliver *separately*.
That is the coverage shape to fix, not just the line.

## Fix direction

Note the distinction the fix must preserve: `childList: true` is required in the **MutationObserver
init** for either subscription — that part is correct and must stay. What is missing is the
**delivery filter**: remember which of `added` / `removed` were subscribed, and drop records the
consumer did not ask for before dispatching.

Check the whole path while you are in it — `mutate-records.ts:170` builds `type: 'children:added'`
and `:195` reads `next.type === 'children:added'`; make sure the filter is applied at one place, not
duplicated at each.

Do not change the public option names or the event payload shape. This is a bug fix, not a redesign.

## Version

**0.2.1** — the version already sitting unpublished in the tree. It was docs-only; it becomes a
docs + correctness patch. Do **not** bump beyond it (owner rule 2026-09-16: the MAJOR never moves,
patch by default). Update the existing `[0.2.1]` CHANGELOG section to lead with this fix — the
heading is already dated and the release-state gate is already green; keep it that way.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **The test that was missing**, failing first: subscribe to `children:added` only, remove a child,
   assert **no** callback. Then the mirror: subscribe to `children:removed` only, add a child,
   assert no callback. Then both-subscribed still receives both.
2. **Negative control:** restore the collapsed condition, watch exactly those tests go red. Put the
   literal output in your report — a test that stays green when the bug is restored is a fake gate.
3. The existing 241 tests stay green; report the real number.
4. `data-observe-state` and the event payload must be unchanged for a consumer subscribing to both —
   prove it, since that is the default-configuration path (standing rule: verification covers the
   default, not only the option under test).
5. A real-browser check is **not** required here — this is subscription bookkeeping, observable in
   jsdom. Say so explicitly rather than claiming browser verification you did not do.
6. Do not run `npm publish`. Leave the work in the tree; do not commit.
