# AUDIT-1 — **P0, blocks everything.** Does each library actually work?

Owner, 2026-09-06:
> "There's something worse, when these are all not working, you verified the work to be done."
> "Only v-fit-children works well and that is taken care of by another agent."

**Read that as the specification for this ticket.** Every library that had tickets run through it
this session is now in a worse or unproven state. The one library nobody touched is the one that
works. That is not a coincidence — it is what happens when verification proves a spec rather than
a product.

## What went wrong, so this ticket does not repeat it

Five tickets reported "browser-verified" with measurements, mutation tests and negative controls.
All of it was **self-verification against the ticket's own description**. Nobody drove the library
as a user, and nobody checked the default configuration:

- **TT-7** proved fit-based `flip` works with `flip: true` and an explicit `placement`. The default
  (`placement: 'auto'`, `flip` omitted) never runs the fit test at all — see TT-15. 9 of 12 cards
  use that default.
- **DZ-1a/d** proved click-to-pick works, and silently broke `pasteOn: 'host'` — see DZ-2. Both
  features pass their own tests in isolation.
- **DZ-1f** edited all 12 dropzone cards and died before verifying any of them, twice.
- **TT-3** landed hide-by-default; its browser pass never ran.
- **COPY-4** proved card 13. Nobody checked the other 12 `v-copy` cards still work.

## The rule for this audit

**You are not given the tickets.** You are given the playground and the packages. Use each library
the way a developer who just installed it would: the **bare binding, no options**, then the
documented recipes from the README. Judge whether it works, not whether it matches an intent.

If you need to read a ticket to know what "working" means, that itself is a finding — the demo and
the README are supposed to tell you.

## Owner's own read, 2026-09-06 — use it to prioritise, not to skip

> "v-copy is okay, dropzone has demo problems but I believe functionality works, we need to confirm
> with smoke tests tho."

So: **`v-teleport-to` is confirmed broken** (TT-15). **`v-dropzone`'s library is probably fine and
its demos are suspect** — 12 cards were edited and never verified. **`v-copy` is reported working**,
though nobody has checked the 12 cards other than 13 since `dedupe` landed. `v-select-text`,
`v-observe` and `v-scroll-into-view` have had no independent pass at all and no browser spec exists
for the last two.

Prioritise: `v-teleport-to` → `v-dropzone` → `v-select-text` → `v-observe` →
`v-scroll-into-view` → `v-copy` → `v-fit-children`. **Confirm rather than assume** the owner's read
— he is describing symptoms, and the point of this ticket is to find causes.

## Scope — one library per run, all seven

`v-teleport-to` · `v-dropzone` · `v-copy` · `v-select-text` · `v-observe` · `v-scroll-into-view` ·
`v-fit-children` (include it: the owner says it works, so it is the control — if your method cannot
confirm a working library, your method is wrong).

For each:

1. **The bare binding first.** `v-thing="minimum"` with nothing else, in a realistic layout. Does
   it do the obviously-right thing? This is the configuration almost every consumer ships and the
   one nobody tested this session.
2. **Every card on the tab, driven in a real browser.** Not "does it render" — click the controls,
   scroll the containers, type in the inputs, and read state back out of the live DOM. Cards whose
   controls do nothing are findings. Cards whose prose contradicts their behaviour are findings.
3. **Feature interactions.** The two worst bugs this session were pairs of features that each
   worked alone (`clickToPick` × `pasteOn`, `flip` × `maxHeight` clamping). For each library, list
   the option pairs a consumer would plausibly combine and try them together.
4. **The README's own recipes**, pasted and run. Every one. `v-select-text`'s README currently
   recommends a `navigator.clipboard.writeText` call that is refused in Firefox and Safari on the
   package's default trigger — that is the class of thing this step catches.
5. **Edge geometry**, since three of these libraries are layout-shaped: reference near a viewport
   edge, inside a scrolled container, in a small boundary, at mobile width.

## Deliverable

A per-library verdict — **works / partly works / does not work** — with, for each finding: what you
did, what you expected, what happened, and a screenshot or a DOM read-back. Rank by whether a
consumer hits it on the default path.

**Do not fix anything.** Findings only. A fix inside an audit is how an audit stops being one.

Explicitly say, per library, whether you would be comfortable publishing it today. `@ozjsey/v-copy`
and `@ozjsey/v-dropzone` are queued for publish (PUB-1) and this ticket gates both.

## Standing rules that apply

- `BOARD.md` acceptance criteria: read state back out of the live DOM; a red check beats a missing
  one; never report as passing what was not exercised.
- The harness itself lies in two known ways: `pnpm interactions` prints a self-selected denominator,
  and `cdp.mjs:149` sends `userGesture: true`, faking activation for gesture-gated APIs. Use
  `Input.dispatchMouseEvent` / `Input.dispatchKeyEvent` for anything that needs a real gesture.
- `CONVENTIONS.md`: look at a screenshot before calling anything fixed.
