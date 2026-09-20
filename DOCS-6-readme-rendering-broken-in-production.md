# DOCS-6 — every documentation tab is broken on the deployed site

**Owner, 2026-09-17, testing the live site:** *"`../v-copy/README.md` does not exist, so there is
nothing to render. Every published package owes a README — see `tickets/_STANDARDS.md`. All
documentation pages are off, it also looks very ugly."*

He is reading the app's own error banner. It is not v-copy's fault — v-copy has a README, 20,001
bytes of it. **All ten tabs are broken, in production, right now.**

## Root cause — confirmed

`src/docs.ts`:

```ts
const readmes = import.meta.glob<string>('../../*/README.md', { query: '?raw', import: 'default', eager: true })
```

The glob reaches **out of the playground** into sibling package directories. Locally those exist and
it works. In CI **only the playground repo is checked out** — `../v-copy/` is not there — so the
glob matches nothing, `readmeFor()` returns `null` for every library, and every tab renders the
"does not exist" banner.

This is the same shape as the library aliases, which already solve it: `vite.config.ts:87` disables
source aliasing under `GITHUB_ACTIONS` and resolves from `node_modules` instead. The docs glob never
got the same treatment.

## The fix has a clean source

npm packs `README.md` into every tarball, so in CI the READMEs are already on disk:

```
node_modules/@ozjsey/v-copy/README.md             20001 bytes
node_modules/@ozjsey/v-fit-children/README.md     18389 bytes
node_modules/@ozjsey/v-observe/README.md          18406 bytes
node_modules/@ozjsey/bigdecimal-string/README.md   8826 bytes
```

Mirror the alias strategy: glob the sibling sources when they exist (local dev — the working tree
is the point), and `node_modules/@ozjsey/*/README.md` otherwise (CI — the published file, which is
also *exactly what npm shows*, which `src/docs.ts`'s own header says is the goal).

Both globs must be static string literals — Vite requires that — so glob both and choose at
runtime; do not build the pattern from a variable.

Keep the failure banner. It is correct behaviour for a package that genuinely has no README; it was
simply firing for the wrong reason. Consider making it name which source it looked in, so the next
occurrence is diagnosable from the screen.

## Second: DELETE the `status` line — it is internal chatter on a public page

**Owner, 2026-09-17**, reading his own site: *"Published — 2.3.0 on npm (owner ozjsey), verified
against registry.npmjs.org 2026-09-17; 2.3.1 built locally, not yet published"* — *"What's this lol
why it's needed, please remove. It's of course my packages there."*

He is right, and it is the better fix. That string is **release bookkeeping addressed to us**, on a
page addressed to strangers: who owns the scope (obvious), when we last checked the registry (nobody
cares), what is built but unpublished (actively confusing — it advertises something they cannot
install). Every one of the ten is also **wrong** after tonight's publish wave, and v-select-text's
names `v2.1.0`, a version that has never existed anywhere.

**Remove the `status` field from the public view entirely.** Do not derive it, do not reword it —
a version number the site must keep true is a maintenance burden that has already failed once, and
the README rendered beside it carries npm version badges that update themselves.

Delete the field from `LibraryManifest` and from all ten manifests if nothing else consumes it; if
a gate or script reads `status`, say so in your report and remove only the rendering, leaving the
data with a comment naming its one remaining reader.

## Third: the rendering is not nice to read

The owner also said it *"looks very ugly."* `src/styles.css` has ~24 markdown rules, so it is not
unstyled — but he was looking at ten error banners, so treat the styling verdict as provisional:
**fix the content first, look at the result, then make a judgement.** Sensible measure once real
READMEs render: a comfortable reading measure (~65-75ch), heading rhythm and spacing that make the
document scannable, table and code-block treatment that survives long lines (they must scroll in
their own container, never the page), and dark/light both checked. Do not redesign the whole app.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **A check that would have caught this**, because nothing did: assert that every library tab
   renders a non-empty README **with the sibling directories absent** — i.e. reproduce CI
   conditions rather than trusting the local tree. Without this the bug returns the next time
   someone changes the glob. **Negative control:** point the fallback at a directory with no
   READMEs and watch the check go red.
2. `pnpm build` with `GITHUB_ACTIONS=1` set and the siblings temporarily renamed/hidden — every
   tab must show content, not a banner. Verbatim output in the report.
3. The derived status line must match `npm view <pkg> version` for all twelve; show the table.
4. Existing gates stay green: `pnpm typecheck`, and `pnpm deeplinks` (README links are followed by
   that gate — changing the README source must not break it).
5. Screenshots of one documentation tab before and after, and in both themes, with absolute paths.
6. Do not commit. Leave the work in the tree.
