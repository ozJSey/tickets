# DOCS-7 — short READMEs, because the playground is the reference now

**Owner, 2026-09-18, twice.** First: *"our readme's are not consistent, we need one good template
and apply it throughout."* Then, correcting the planner's first draft of this ticket, which was a
full reference template:

> *"Since we have playground now, our readme's should be small, precise (stating the problem and
> solution nicely) installation nicely, some usage examples and kind of it?"*

That is the better division of labour and this ticket follows it. **The playground is the
exhaustive reference** — ten tabs, 144 demo cards, every option driven in a real browser, and now
the package README rendered beside it. The README's job is no longer to be a manual. It is a
landing page: convince, get them installed, show them the shape, hand them to the playground.

Today they run **358 to 773 lines** with twelve different section orders. Target: **roughly 80–150
lines**, one shape.

## THE TEMPLATE

```
# @ozjsey/<name>

<ONE sentence: what it does, in the reader's words.>

[npm badge]

## The problem
  2–4 sentences. The situation the reader is actually in, before any mention of
  this package. Concrete: the thing they hand-rolled, the thing that broke.

## The solution
  2–4 sentences plus the smallest honest code sample. What this does about it,
  and the one behaviour that defines it. If a named incumbent exists, ONE line
  on what it does not do — this is the wedge and it stays.

## Install
  npm install @ozjsey/<name>
  Requires <peer>.          ← the MEASURED floor, not a habit
  <registration snippet, if the package needs one>

## Usage
  2–3 examples, no more. The bare/default case FIRST — the portfolio rule is that
  the bare directive covers 95% of cases. Then one or two that show range.
  Each example does something a real person wants; none is a parameter tour.

## Everything else
  → the playground tab (deep link), stated as what it is: every option, every
    event, driven in a real browser.
  → CHANGELOG.md · ARCHITECTURE.md
```

## What is deliberately GONE from the README

Option tables, event tables, data-attribute tables, recipe collections, type dumps, migration
guides, FAQ. **All of it already exists in the playground, driven and verified**, which a static
table is not. Do not summarise it here; link to it.

## Rules that outrank the template

1. **Cut, do not rewrite.** The prose in these files is good (the audits said so repeatedly).
   Choose the best existing sentences and delete around them. New writing only where a required
   section has nothing to inherit. A README of fresh generic filler is a failed ticket even if it
   is the right length.
2. **Every surviving claim must be checkable against the source.** Audits found false claims in
   five packages — a `.select()` fallback that does not exist, `NaN` handling documented backwards,
   an "Upgrading from 1.x" path to a 2.x never published. Cutting is the chance to remove them:
   **delete anything you cannot verify**, and list what you dropped for that reason separately from
   what you dropped for length.
3. **Known limitations are not deleted, they move.** This portfolio's rule is that failing a test
   beats claiming something we do not do. If a package has a real limitation a reader would hit in
   the first hour, it stays — one line, in "The solution". The rest go to the playground tab.
4. **No version numbers in prose.** They rot. The badge carries the version.
5. **Playground links must be real deep card links** — `pnpm deeplinks` validates all of them and
   fails on an invented one.
6. Touch ONLY `README.md`. Not CHANGELOG, not ARCHITECTURE, not source.

## Planner's one reservation, recorded and overridden

A shorter README moves the reader's first contact with the hard parts — the limitations, the
gotchas — off npm and onto a site they have to click through to. For a package whose defining
behaviour is surprising (`vue-write-behind`: *the server's reply is discarded on purpose*), that
line earns its place on the npm page itself. Rule 3 above is how this is handled: the defining
surprise stays, the catalogue goes. Flag it if you think a specific package loses something real.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. Every README in the 80–150 line band, or a stated reason why that package genuinely needs more.
2. `pnpm deeplinks`, `pnpm markdown`, `pnpm docs:ci` green — report real numbers.
   `pnpm docs:check` no worse than now (fails only on `write-behind`, pre-existing, `PROGRESS.md:81`).
3. **Two separate lists**: claims deleted because they were FALSE (with the source line that
   decides each), and content moved out because the playground covers it. Conflating them hides
   the first inside the second.
4. Do not commit. Leave the work in the tree.
