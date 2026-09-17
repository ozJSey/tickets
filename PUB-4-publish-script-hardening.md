# PUB-4 — harden and test `scripts/publish.mjs`

**Provenance, stated honestly:** the planner wrote this script by hand on 2026-09-16 (the owner
wanted the publish order *"hard coded in the script, in the way it's meant to be"*) while every
subagent was dead on the session limit. Per the working agreement that makes it a **draft** —
only its happy-path plan mode has ever run. This ticket is the hardening it owes.

## What must not change

- `ORDER` stays a hard-coded array — it IS the decision. Engine before adapter
  (`write-behind` → `vue-write-behind`), stable alphabetical otherwise, `vue-provide-seeker`
  absent (VS Code extension). A new package is one added line.
- Three modes: bare = plan (no side effects) · `--check` = plan + pre-flight · `--publish` =
  pre-flight then `npm publish` per package, in order, stop on first failure.
- **BEHIND-the-registry aborts the whole run.** Parity skips. Publishing is only ever run by the
  owner invoking `--publish` — never by an agent, never in CI (BOARD.md decision 5).

## Work

1. Split decision logic from I/O so it is testable: `cmp(a,b)`, `decideAction(local, registry)`,
   and the queue builder become exported pure functions; the CLI stays a thin shell over them.
2. Tests via **`node:test`** (zero new deps, no root node_modules): the version comparator
   (incl. `10.0.0 > 9.0.0`, missing segments), every `decideAction` branch, queue ordering
   (write-behind strictly before vue-write-behind when both are ahead), and — mocked spawn —
   that `--publish` stops at the first pre-flight failure and **never** reaches `npm publish`
   on a dirty tree or a failing test suite.
3. Negative controls, per the standing rule that a gate that cannot fail is worthless:
   registry-ahead fixture → exit 1 with the ABORT message; dirty-tree fixture → exit 1 before
   any package command runs. Tests must be shown red against a deliberately broken comparator.
4. `npm view` failure handling: distinguish 404 (first release → PUBLISH) from network failure
   (→ abort loudly; today both collapse to "never published", which would mis-plan a first
   release during an npm outage).
5. Live validation, no publish: `node scripts/publish.mjs --check` against the real tree must
   come back clean for whatever is genuinely ahead at run time.

## Acceptance

- All tests green via `node --test scripts/`; the two negative controls demonstrably fire.
- Behaviour of the three modes byte-compatible with the draft except where this ticket names a
  change (the 404-vs-network split is the only semantic change).
- No new dependencies anywhere. Agent runs at xhigh effort (standing rule, 2026-09-16).
