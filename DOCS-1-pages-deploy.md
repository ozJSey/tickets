# DOCS-1 — playground + documentation per project, deployed to GitHub Pages

**Owner requirements (2026-09-05):**
> "Every playground item must have a documentation tab, so playground and tab, we will deploy
> these to Github pages and attach them to our library readme"
> "playground and documentation in each project."

So: **each project gets two views — Playground and Documentation** — published to GitHub Pages
and linked from that package's README. Remote already exists:
`https://github.com/ozJSey/os_ideas.git`.

This turns the playground from a private harness (`package.json: "private": true`, and
`instructions/playground.md` still describes it as never-published) into a **public artifact**.
Everything in `tickets/_STANDARDS.md` applies with more force, because the failure mode is now
visible to strangers.

---

## The deploy target is a SEPARATE repo

Owner, 2026-09-06: the playground's origin is `https://github.com/ozJSey/ozjsey-npm-playground`
(renamed from `open-source-docs`). It is **not** `os_ideas`, where the playground currently lives
as `npm/playground/`.

**This breaks the playground's core mechanism if handled naively.** `playground/vite.config.ts`
aliases all seven libraries by *relative path* (`pkg('../v-copy/vCopy.ts')`), which is what gives
HMR when a library source is edited and what makes the playground double as a smoke test of the
real source. In a separate repo those siblings do not exist.

Three options — **recommendation is (3)**:
1. **Install the packages from npm.** Honest consumer's view, but source HMR is gone and an
   unpublished package cannot be demoed at all — which today is 6 of the 7 v-* libraries plus both
   new ones.
2. **Git submodule.** Keeps sources; adds a submodule step to every clone, every CI run, and every
   contributor's first five minutes.
3. **Build in `os_ideas`, deploy the built artifact to `ozjsey-npm-playground`.** Development is
   unchanged — relative aliasing, HMR, `PLAYGROUND_TARGET=dist` all keep working. The other repo
   holds only static output and serves Pages from it. No submodule, no publish dependency, source
   of truth stays with the packages. The deploy is a CI step that pushes `dist/` to that repo.

Whichever is chosen: **`base` must be `/ozjsey-npm-playground/`** (or a custom domain), and every
hash link, asset URL and service-worker scope has to agree with it.

## BLOCKER — 6 of 12 `v-dropzone` cards break on a static host

`playground/vite.config.ts:77-93` serves `/api/upload`, `/api/upload-slow` and
`/api/upload-fail` from a **`configureServer` middleware**. That hook runs in `vite dev` only —
`vite build` emits no server. On GitHub Pages those routes 404.

Cards that depend on them: `04-paste`, `05-url-upload`, `07-api`, `08-auto-upload-queue`,
`09-css-progress`, `10-state-machine` (plus the manifest's prose).

**This must be solved before the first deploy, not after.** Options, with the recommendation:

- **(A) Service Worker intercepting `/api/upload*` — recommended.** Works on a static host,
  and it keeps the *real* code path under test: real `XMLHttpRequest`, real `upload.onprogress`
  events, real abort. Costs: SW registration under the Pages subpath, a scope/`base` interaction,
  it must not interfere with `pnpm dev` or the test harness, and the first load has a
  registration race the demos must tolerate.
- **(B) Rewrite those cards to function-based upload** (`upload: async (file, signal, onProgress)`).
  No server needed — but it stops demonstrating URL upload, which is a documented feature with
  its own README recipe. **This makes the published docs quieter about a real capability.** Only
  acceptable if (A) proves unworkable, and then the cards must say so out loud.
- **(C) A public echo endpoint** — rejected: external dependency, CORS, rate limits, and the
  docs break when someone else's service does.
- **(D) A build-time swap to a client-side fake, dev only having the real path** — rejected.
  That is the "the demo does not do what it says" failure this repo has hit three times.

## Second blocker — base path

No `base` in `playground/vite.config.ts`. See the deploy-target section above: it must match
`ozjsey-npm-playground`, not `os_ideas`.

---

## Design questions to settle

**1. Where does documentation content come from? Recommendation: the package README, rendered.**
Do **not** author docs twice. This repo has documented, repeated drift — `v-select-text`'s README
described `condition` for six runs after the source moved to `enabled`; `CLAUDE.md` still claims
`v-fit-children` is 2.2.0 when the table says 3.0.0; it also still says there is no remote.
Two hand-maintained copies of the same API docs will diverge, and now they diverge *in public*.

The README is already the npm-facing documentation and ships in the tarball, so make it the
single source: the Documentation view renders `<package>/README.md`. Per-card documentation
comes from the manifest, which already carries `blurb` and `tags` — extend it only if a card
genuinely needs more than that, and keep it in the manifest so the card and its docs cannot be
moved apart. Decide and justify: markdown rendering needs a dependency (the playground's
zero-dep rule is lifted, so this is allowed — pick a small one and say why).

**2. Deep links.** READMEs must link to a specific library and ideally a specific card, so
`syncFromHash` has to accept `#<lib>/<file>`. The COPY-1 design already flagged this as its
"T5b". Settle the URL shape once, here, because every README link depends on it and links that
ship in a published README are expensive to change.

**3. Sources or dist for the published build?** The playground aliases package *sources* by
default and has a `PLAYGROUND_TARGET=dist` mode. **Recommendation: publish the `dist` build** —
a public docs site should show the consumer's view, and it makes a stale `dist` publicly
obvious rather than silently hidden. `CLAUDE.md` records `dist/` going stale three separate times.

**4. This introduces the first CI in the repo** (`ls .github` → nothing). The deploy workflow
must **gate on the test suite before publishing**, per `_STANDARDS.md`. Deploying documentation
that demonstrates broken behaviour is worse than not deploying. Coordinate with PG-11: if the
Playwright migration lands, this workflow is where its gate runs.

**5. The coverage ledger becomes public.** Once deployed, "59 of 83 cards have no check" is a
statement about a public site. That strengthens PG-11's ledger work; it does not block this.

---

## Acceptance

- Each of the 9 library tabs (7 today + 2 new) has a working Documentation view alongside its
  Playground view.
- Every one of the 6 upload-dependent cards works on the deployed static site, with the same
  code path as locally — or, if (B) was chosen, says plainly what it no longer demonstrates.
- Each package README links into the deployed site, to its own library, using the settled URL shape.
- `pnpm build` output loads correctly under the Pages base path.
- The deploy workflow runs the test gate first and refuses to publish on failure.
- `playground/package.json` stays `"private": true` — this is a Pages deploy, not an npm publish.
- `instructions/playground.md` and `playground/README.md` updated: the playground is no longer
  "private, never published".
