# DOC-1 — three shipped tarballs lie about their own release state; sweep all twelve

**Owner's standing rule, restated 2026-09-16:** my job is *"checking if it matches requirements and
we don't lie in changelogs."* Three confirmed violations are live on npm right now — each verified
by the planner against `registry.npmjs.org`, not against repo files.

| Package | Registry | Its own SHIPPED CHANGELOG/README says |
|---|---|---|
| `@ozjsey/v-fit-children` | `2.2.0`, `2.3.0`; **latest = 2.3.0** | CHANGELOG:25 *"[2.3.0] — unreleased / Built locally, **not published**. npm's `latest` is 2.2.0"*; README:399 same |
| `@ozjsey/v-observe` | `0.1.0`, `0.2.0` both live | CHANGELOG:16 *"0.1.0 was never published"* |
| `@ozjsey/bigdecimal-string` | `1.2.0` published 2026-09-14T10:01:33Z | CHANGELOG:13 *"[1.2.0] — UNRELEASED / Not on npm"* |

Each of these files is inside the published tarball, so **the npm page for the released version
tells every visitor that the version they are reading does not exist.** This is the workspace's own
standing rule ("verify publish state against the registry, never against a repo file") failing in
the one artifact strangers actually read.

## Work

1. **Sweep all twelve publishable packages**, not just the three. For each: fetch
   `npm view <name> versions dist-tags` **and** the published tarball, and diff every release-state
   claim in the packed `CHANGELOG.md` / `README.md` against the registry. Report the full table —
   packages that are clean get a row saying so.
2. Correct every false claim: real release date on the heading, drop "unreleased / not published"
   paragraphs, fix any "npm's latest is X" line that is wrong.
3. **Fix the generator of the defect, not just the instances.** These files were written while the
   version was genuinely unpublished and never revisited at publish time. Add a gate:
   `scripts/publish.mjs` (see PUB-4) refuses to publish a package whose packed CHANGELOG contains
   an unreleased/not-published marker for the version being published. A pre-publish check is the
   only thing that stops this recurring — it has now happened three times.
4. Propagate: `CLAUDE.md` and `instructions/v-fit-children.md` repeat the stale
   "2.3.0 not published" claim; correct them in the same pass.

## Explicitly NOT in scope

The dead `repository`/`homepage`/`bugs` URLs on published `v-observe` 0.2.0 and
`v-keyboard-navigation` 0.3.0 (both 404 anonymously). Local package.json files were corrected in
`19d2c7d`, but **the published artifacts still carry the dead URLs** — that only clears on the next
publish. That is META-1 and remains the owner's call; note it in the completion report, invent no URLs.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. The twelve-row table above, every cell sourced from the registry + the packed tarball, not from
   repo files.
2. The publish gate from step 3 lands with a **negative control**: a fixture CHANGELOG carrying an
   "unreleased" marker must make the gate refuse, demonstrated failing.
3. No version bumps in this ticket beyond what the corrections ride on — the fixes publish
   alongside the next patch each package is already taking (v-fit-children with FIT-2,
   bigdecimal-string with BD-1). `v-observe` needs its own patch; use one.
