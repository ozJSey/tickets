# DG-1 — `@ozjsey/dependency-grouper` is demonstrated by the playground using it

Owner, 2026-09-13:
> "We have 11 projects, 10 is presentable, 11 (dependency-grouper) can just be the way we implement
> playground."

So it is exempt from having a demo *card*, and its demonstration is that **this repo actually uses
it**. Dogfooding as documentation. Published, `@ozjsey/dependency-grouper` 0.3.5, live.

## The tension to solve, not to work around

The tool groups shared dependency sets across **workspaces** — pnpm, npm, yarn. This repo has
**none**: no root `package.json`, no `pnpm-workspace.yaml`, and `CLAUDE.md:5` states plainly that it
"is **not** a monorepo and there is no workspace root." Twelve sibling folders each carrying their
own `node_modules`.

That is either the blocker or the point, and the ticket is to find out which:

- **If the tool can serve a non-workspace set of siblings**, this repo is its best possible
  advertisement — twelve projects with heavily overlapping devDependencies (vitest, tsup,
  typescript, vue) and no shared root. Make it work here, and the README gains a real case.
- **If it genuinely requires a workspace**, that is a documented limitation the README does not
  currently state, and finding it by trying is worth more than a card would have been. **Report it
  as a finding and say what the tool would need.**

Do not fabricate a workspace just to make the demo work. If a root workspace is the right answer for
this repo on its own merits, say so as a separate recommendation with its costs — it would change
how every package installs and tests.

## What to build

1. **Use it for real.** Run it against these packages, group what is genuinely shared, and commit
   the result. The proof is the diff.
2. **A short section in the playground's documentation view** — not a demo card — explaining that
   the repo's own dependency sets are managed by this tool, with a before/after of one real
   `package.json`. `DOCS-1` owns the docs mechanism; if it has not landed, build the content so it
   drops in.
3. **Its README gains the case study**, since it is published and the README is the npm page.

## Standards gaps in the package itself

- **Zero automated tests** — a 657-line hand-rolled `test/test.js`, and it has 13 published
  versions. The owner's instruction on this (2026-09-06) was: add tests, **but trust the code over
  the tests** — 13 versions in the field are validation the tests do not have. Write
  **characterisation** tests that pin what it already does. A failing test means the test is wrong
  until proven otherwise; do not "fix" behaviour to satisfy a new assertion.
- No `ARCHITECTURE.md`, no `CHANGELOG.md`. Both required by `tickets/_STANDARDS.md`; the changelog
  is a publish blocker and this package is published.
- It was **not** covered by the 9-package quality audit. File anything structural you hit rather
  than fixing it silently in this ticket.

## Acceptance

- The repo's own packages are grouped by the tool, or a documented reason why they cannot be.
- The playground's docs carry the case study with a real before/after.
- Characterisation tests exist and pass; the count is recorded.
- `ARCHITECTURE.md` and `CHANGELOG.md` exist.
- Nothing about the tool's behaviour changed to make the demo easier.
