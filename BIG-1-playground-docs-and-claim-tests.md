# BIG-1 — `bigdecimal-string`: playground, docs, and a test per claim

Owner, 2026-09-13:
> "We need big decimal documentation too, ideally with and withouts for our claims, of course e2e
> test everything make sure what we claim is always visible."

Three things: a playground tab, a documentation view, and **an end-to-end test per claim that
asserts the claim is visible on screen** — not merely that the maths is right.

## Why with-and-without is the whole design

This library's value proposition is **invisible on its own.** `"0.30"` means nothing until it sits
beside `0.30000000000000004`. Every card is therefore two columns:

```
        plain JavaScript                 bigdecimal-string
0.1+0.2  0.30000000000000004              0.30
```

The README already has this shape — a "The Problem" / "The Solution" pair using `0.1 + 0.2`. It
asserts the contrast in prose. The tab must **perform** it, computing both halves live in the
browser rather than printing a hardcoded string.

**Compute the native answer at runtime.** A hardcoded `0.30000000000000004` is a claim about
JavaScript, not a demonstration of one, and it cannot rot in a way anyone would notice.

## The tab

**This is the first non-directive tab.** The registry globs `src/demos/*/manifest.ts` and nothing
requires a Vue directive, but check `libraries.ts` and `DemoCard.vue` for anything that assumes one
— `PG-3` and `COV-1` flagged this as the open structural question and this ticket is where it gets
answered. Do the smallest thing that makes non-directive packages first-class, not a special case.

Card set, one per claim, each with the two columns:
- **The REPL** — two operand inputs and an operation picker, both answers side by side, live. This
  is the headline card; `TASKS.md` has called it the portfolio's highest-value missing demo since
  2026-08-16.
- Each remaining claim in **"The Problem"** and **"Features"** gets a card: precise decimals via
  BigInt, formatting/thousand separators, the rounding modes, chainable arithmetic, comparisons,
  large-number display, and the currency case.
- **Enumerate the README's claims first and map each to a card.** A claim with no card is a gap to
  report, not to quietly skip.

## The tests, which are the point

For every claim: an end-to-end check that **both halves are rendered and correct**.

- Assert the library's output **and** the native output are present in the DOM, with the expected
  values. If the contrast ever collapses — the library regresses, or a card stops rendering one
  side — the check fails.
- **Assert visibility, not just presence.** "Always visible" is the owner's phrasing: an element
  that exists but is clipped, `display:none`, or scrolled out of a collapsed panel does not
  demonstrate anything. Check the rendered box, not just `textContent`.
- Drive the REPL with **real typed input** (`Input.dispatchKeyEvent`), not synthetic `input` events.
- **Negative-control the suite.** Break the library's output and confirm the matching check goes
  red. Several agents have found their own checks passing vacuously; the ones that ran controls
  found it, the ones that did not shipped the illusion.
- Add `playground/scripts/interactions/bigdecimal-string.mjs` and raise the coverage ratchet.

## Standards gaps to close in the same run (`tickets/_STANDARDS.md`)

- **No `ARCHITECTURE.md`** — one of only three packages without one. `src/` is already split
  (`big-decimal.ts`, `types.ts`, `utils.ts`); name the module map and the invariant.
- **No `CHANGELOG.md`** — and this package is **published at 1.1.0**, so it needs one before any
  further release. `npm view bigdecimal-string time` gives the publish dates to reconstruct from.
- A **documentation view** per `DOCS-1`, rendered from the README rather than authored twice.

## Notes

- **Published, 1.1.0, unscoped.** The README is the npm package page. Everything it claims is
  public today.
- **122 tests, and no browser check of any kind.** The maths is well covered; nothing has ever
  confirmed a user can see it.
- This package was **not** in the 9-package quality audit. If the reviewer's eye catches structural
  problems while working here, file them — do not fix them silently in a docs ticket.

## Acceptance

- Every README claim has a card, or is reported as an unmapped claim.
- Every card shows the contrast computed live, both sides.
- Every claim has an e2e check asserting both halves are visibly rendered with the right values.
- Negative controls prove the checks can fail.
- `pnpm typecheck`, `pnpm smoke`, `pnpm smoke:dist` green; screenshots reviewed.
- `ARCHITECTURE.md` and `CHANGELOG.md` exist.
