# PUB-6 — the publish gate must catch an adapter that cannot reach its engine

**Owner, 2026-09-17**, on being told `vue-write-behind` 0.2.0 shipped while `write-behind` stayed
0.1.0: *"this writebehind would have a feature to use WebWorkers, if we updated writebehindvue and
not writebehind it concerns me."* The instinct was right, and the failure is silent.

## The defect class

`vue-write-behind` declares `"@ozjsey/write-behind": "^0.1.0"`. On `0.x` versions a caret pins the
**minor**, so the range is `>=0.1.0 <0.2.0`. Measured with the real semver:

```
engine 0.1.1  satisfies "^0.1.0"?  YES
engine 0.2.0  satisfies "^0.1.0"?  NO  <-- would NOT be installed
engine 1.0.0  satisfies "^0.1.0"?  NO
```

WBC-8 ships the WebWorker host as engine **0.2.0**. If the adapter's range is not widened in the
same wave, every Vue consumer keeps resolving a 0.1.x engine: the feature is on the registry, the
install succeeds, no warning is printed, and the capability simply is not there. Nothing in the
current pre-flight would notice — it checks versions against the registry and docs against release
state, never one package's declared range against a sibling's shipped version.

This is not hypothetical bookkeeping: it is the same shape as tonight's peer-range defect
(`vue ^3.0.0` declared against 3.2-only APIs), where the manifest promised something the artifact
could not deliver, and four packages shipped it.

## Work

Add a pre-flight step in `scripts/lib/release.mjs`, beside the existing changelog gate, and a pure
function beside it in `scripts/lib/` so it is testable without a network (keep PUB-4's structure).

For every package in the queue, for each of its `dependencies` **that is also a package in
`ORDER`**:

1. Resolve the sibling's version that will exist after this run — its local `package.json` version
   if the sibling is also queued, otherwise its published `dist-tags.latest`.
2. Assert the declared range **admits** that version. If it does not, refuse with a message naming
   both packages, the range, the version, and the fix — widening the range, not bumping the engine
   back down.
3. Run it in **both directions**, because both are real:
   - an adapter whose range cannot reach the engine being shipped (this ticket's case);
   - an engine being published that no queued adapter can reach, which is the same defect noticed
     one package earlier.
4. Use the `semver` implementation already present rather than hand-rolling range logic — a
   hand-written caret parser is exactly the kind of thing that is subtly wrong on `0.x`.

Only `vue-write-behind → write-behind` exists today; write the check generically over `ORDER` so
the next pair is covered without anyone remembering.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. Tests alongside `scripts/publish.test.mjs`, with a mocked runner — no network:
   - adapter `^0.1.0` + engine 0.2.0 queued → **refuses**, message names both packages
   - adapter `^0.2.0` + engine 0.2.0 queued → passes
   - adapter `^0.1.0` + engine 0.1.1 queued → passes (the patch case that is fine today)
   - engine queued alone while an unqueued adapter's published range excludes it → refuses
   - a package with no sibling dependencies → unaffected
2. **Negative control:** disable the check, re-run the first case, show it passing — proving the
   gate is what catches it and not some other assertion. Literal output in the report.
3. `node --test 'scripts/*.test.mjs'` green, real numbers reported.
4. Run `node scripts/publish.mjs --check` against the real tree afterwards: everything is at parity
   tonight, so it must stay clean. A gate that fires on a healthy tree is worse than no gate.
5. Do NOT run `npm publish`. Mock it.
