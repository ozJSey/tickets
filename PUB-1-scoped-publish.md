# PUB-1 — publish under the `@ozjsey` scope

**Three packages now:** `@ozjsey/v-copy`, `@ozjsey/v-dropzone`, `@ozjsey/v-scroll-into-view`
(added 2026-09-06: *"We can publish scrollinto view as well, it's just there.. Nice to have, won't
hurt, sure it's easy.."*). All three scoped names verified **free**; `npm whoami` → `ozjsey`.

Owner, 2026-09-06: *"Let's publish v-copy and v-dropzone, but they need to have @ozjsey/ prefixed
in the names."* This settles NPM-1 in favour of scoping, and matches `CONVENTIONS.md`, which
already said packages publish under the scope.

Verified 2026-09-06: `@ozjsey/v-copy` and `@ozjsey/v-dropzone` are both **free**; `npm whoami` →
`ozjsey`. Unscoped `v-copy` is squatted (egoist 0.1.0, 2017); unscoped `v-dropzone` is free but
being abandoned in favour of the scope.

## Readiness — read before publishing

**`@ozjsey/v-scroll-into-view`: the strongest candidate, with one gate.** 236 tests across 5
projects, untouched since 2026-08-09 so nothing recent destabilised it, 353-line README. But it has
**zero interaction specs — it has never been driven in a browser.** It is a scroll and layout
library, the one category where jsdom is weakest: no layout, no scrolling, no meaningful
`scrollIntoView`. 236 green jsdom tests here is the same confidence that was wrong about
`v-teleport-to` this morning.

Gate: **one AUDIT-1 pass**, plus a `dist` rebuild — the artifact is from 2026-08-09 and
`CLAUDE.md` records `dist/` going stale silently three separate times. Then publish.

Fix in the same pass, found by the KB-1 survey and assigned here: **pairing `scrollIntoView` with
focus requires `preventScroll: true`, or it silently does nothing** — the browser scrolls first, so
`block: 'nearest'` then finds the element already visible and no-ops. Measured in Chrome. The README
does not mention it, and a consumer combining this package with focus management hits it blind.
Version → **1.2.0** for the docs fix, or 1.1.0 unchanged if the fix is docs-only; recommend and
confirm.

**`@ozjsey/v-copy`: ready.** 61 tests; `dedupe` browser-verified with two negative controls; card 13
driven on source and dist. Gaps that do **not** block a first publish but should be recorded:
no permanent `scripts/interactions/v-copy.mjs` (COPY-4 recommended the checks but did not land
them), and no documentation view yet (DOCS-1).

**`@ozjsey/v-dropzone`: NOT ready. Do not publish until DZ-2 resolves.**
1. `pasteOn: 'host'` is a documented option with its own README recipe, and the click-to-pick
   default may have broken it — clicking the zone now opens a file dialog instead of focusing it,
   and nobody has measured whether ⌘V still reaches the host listener. **Open bug in a shipped
   feature.**
2. All 12 demo cards are committed but were **never browser-verified** — the agent died at that
   step twice.
3. `scripts/interactions/v-dropzone.mjs` has a known failing check (stale `display: none`).

Publishing it now contradicts the owner's own standard: *"people use libraries to use confirmed
code, not for not working stuff."* Escalate rather than proceed.

## BLOCKING — the README is the documentation

Owner, 2026-09-06: *"They need to have great readme's as we don't have any documentation
published yet."* Until DOCS-1 ships Pages, **the README is the only documentation a user will
ever see**, and npm renders it as the package page. Treat it as the product surface, not a file.

Measured 2026-09-06: `v-copy` 275 lines / 21 headings / 17 code blocks / **0 badges**;
`v-dropzone` 602 / 31 / 22 / **0 badges**.

**`v-copy` — the lead is wrong, and this is the whole point of publishing it.** The COPY-1 survey
found that **no Vue or browser clipboard library ships history** (`keywords:copy-history` → 0
packages; VueUse `useClipboard` has none), while aggregated copy is replicable by any incumbent in
one line. Yet "Keep a copy history" is currently the *second item under Usage*. The positioning
rewrite was T10 of that design and never ran. Required before publish:
- Title + one line: *"Vue 3 directive that copies any element — and keeps the last N copies so your
  users can pick one back out."*
- First code block stays the 90% case, so a reader can paste something in five seconds.
- **Immediately after: the history-picker recipe** (~15 lines, reactive controller +
  `v-teleport-to` dropdown), linking playground card 13.
- Feature list reordered: history + `dedupe` first, `textContent` source second, feedback / a11y /
  fallback after. Aggregated copy moves to a **Recipes** section, snippet kept verbatim.
- A short, checkable **"Why not VueUse `useClipboard`?"** table — `copy()`/`copied`/`text` versus
  `sink`/`max`/`dedupe`/`rich`/re-copy — saying plainly that VueUse is right when you only need to
  write once. An honest comparison beats a vague superiority claim.

**Both packages:**
- npm version + license + bundle-size badges (sizes are measured and current: v-copy 2.94 KiB
  gzipped, v-dropzone 15.1 KiB / 5.2 KiB gzipped).
- Install line uses the **scoped** name.
- Every code sample imports the scoped name.
- A "part of a set" cross-link section — this matters more than usual here, because per
  `CLAUDE.md` most people copy the source rather than install it.
- No claim that is not true today. `v-dropzone`'s README documents `pasteOn: 'host'`, which DZ-2
  may have found broken — **do not publish a README describing behaviour nobody has verified.**

## The rename, per package

1. `package.json`: `"name": "@ozjsey/<pkg>"`, and **add `"publishConfig": { "access": "public" }`** —
   scoped packages default to `restricted` and the publish will otherwise fail or go private.
2. Version: `@ozjsey/v-copy` → **1.1.0** (keeps the CHANGELOG honest; it already documents
   1.0.0 → 1.1.0). `@ozjsey/v-dropzone` → **1.0.0**, not 0.1.0 — it has 309 tests, folder drops,
   uploads, a programmatic API and full a11y; `0.1.0` would misrepresent it. `v-dropzone/TASKS.md`
   already lists the 1.0.0 bump as its publish-prep item. **Confirm both with the owner.**
3. README install lines, and every `import … from 'v-copy'` in code samples.
4. Playground wiring — all three must change together or the tab breaks:
   `playground/tsconfig.json:24-25` (paths), `playground/vite.config.ts:16-17` (aliases),
   `playground/src/libraries.ts:19-20,29-30` (imports + the registry map).
5. Demo `.vue` files that import the bare specifier.
6. `CLAUDE.md` and root `TASKS.md` status tables — both currently say these are unpublished.
   `CLAUDE.md`'s "only v-fit-children, bigdecimal-string and dependency-grouper are live" line
   becomes wrong the moment this lands.
7. `DIRECTIVE_NAME` does **not** change — `v-copy` stays the directive name; only the package
   moves. Same for the plugin exports.

## Publish

```
npm run build && npm test && npm pack --dry-run   # confirm dist-only, no source/test leakage
npm publish --access public
```
Then verify from a clean directory: install the tarball, import it, and confirm the plugin
registers — do not trust the publish output alone.

## Acceptance

- `pnpm typecheck`, `pnpm smoke`, `pnpm smoke:dist` green **after** the rename — the playground
  resolves packages by specifier, so a missed alias fails loudly rather than silently.
- The published tarball contains only `dist` + README + LICENSE + package.json.
- A fresh-directory install of the published package works.
- Docs updated so no file still claims these are unpublished.
