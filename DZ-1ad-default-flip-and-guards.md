# DZ-1a/d — flip `clickToPick` to default-on, behind one resolver, with stray-click guards

Design and reasoning: `DESIGNS.md` → "DZ-1". Standards: `tickets/_STANDARDS.md`.
**Unblocked** — DZ-1b/c landed: the picker input is now visually hidden rather than
`display: none`, focusable, named, and its focusability follows the affordance. 283/283 tests,
20/20 browser checks. So `main` can now take the default flip without shipping an inaccessible
click target.

Owner: *"v-dropzone click by default should open file/image whatever format selected picker.
It should be something to opt out."* Package is unpublished (`npm view v-dropzone` → 404), so
the breaking default costs nothing and needs no alias.

## DZ-1a — the flip, and the silent bug it exposes

`src/directive.ts:70` reads `if (opts.clickToPick)`. `:116-117` reads
`const wasPicker = !!existing.opts.clickToPick` / `wantsPicker = !!next.clickToPick`.
Flip only `:70` and those two lines read "omitted" as **off** while attach reads it as **on** —
two silent failures on exactly the migration a consumer performs:
- `{ clickToPick: false }` → `{ on }`: falls to the `existing.pickerInput` branch at `:126`,
  which is null because it was never created → **the zone never gains click-to-pick.**
- `{ clickToPick: true }` → `{ on }`: `wasPicker=true, wantsPicker=false` → **teardown runs and
  the zone silently loses it.**

Fix: export `wantsClickToPick(opts) => opts.clickToPick !== false` from `src/picker.ts` and use
it at `:70` **and** `:116-117`. Same idiom the package already uses at `picker.ts:18` and
`directive.ts:98`. Flip the jsdoc default at `types.ts:204`; update the stale comments at
`state.ts:63,65` and the `picker.ts:4` module header.

## DZ-1d — guards, because "click anywhere" is now the default

Add to `onHostClick` (`src/picker.ts:59-69`):
1. **Text-selection guard.** A selection that ends inside the zone fires a `click` — the user
   who just selected the instructional text gets an OS dialog. Bail when `window.getSelection()`
   is non-collapsed with anchor or focus inside the host. **No jsdom test for this** — jsdom's
   Selection is inert (`isCollapsed` always true), so a passing jsdom test would be a lie.
   Browser-only.
2. **`event.detail > 1`** — a double-click otherwise opens two pickers.
3. **`clickIgnore?: string`** (new option) merged into the `closest()` guard, for custom
   clickables the built-in list cannot see. Widen `INTERACTIVE_SELECTOR` (`constants.ts:18-19`)
   with at least `[tabindex]:not([tabindex="-1"])`, `[role="button"]`, `summary`,
   `audio[controls]`, `video[controls]`. The host-itself exemption at `picker.ts:63` must survive
   — a host carrying its own `tabindex` must not exempt itself.

## Known blast radius (verified)

- `vDropzone.test.ts:934` — `default (clickToPick omitted) — clicking host does NOT open a
  picker`: both assertions invert.
- `vDropzone.test.ts:3245-3249` — `api.open() creates a hidden file input on demand`:
  `toBeNull()` fails, the input now exists at mount. The "on demand" premise survives only for
  `clickToPick: false`.
- Add regressions for both `updated`-diff bugs above.
- README: quick start `:50` silently gains click-to-pick (add `cursor: pointer` + a focus ring);
  recipes 1-5 and 8 (`:86,:92,:104,:118,:134,:188`) describe behaviour that is now incomplete;
  recipe 6 (`:142-145`) inverts to "opt out"; recipe 7 (`:152`) reworded; options table `:338`
  default `false`→`true` plus a `clickIgnore` row; API table `:368`; caveats.
  **There is no Accessibility section — add one.**

## Acceptance

- TDD. All 283 tests green plus new ones. The 9 interactive-guard tests at `:995-1094`, the
  reuse guards at `:3314`/`:3348`/`:3351` and the toggle at `:1223` stay untouched and passing.
- `npm run build`, `npm pack --dry-run` clean. From `playground/`: `pnpm typecheck`, `pnpm smoke`,
  `pnpm smoke:dist` green.
- **Browser-verify** (owner rule; and note jsdom cannot see guard #1 at all): drive over CDP via
  `playground/scripts/lib/cdp.mjs`. Prove: a bare zone opens the picker on click; the selection
  guard suppresses it after a drag-select ending in the zone (`Input.dispatchMouseEvent`
  down→move→up); a double-click opens exactly one; a `clickIgnore` match and its nested child are
  both suppressed; `clickToPick: false` opens nothing and adds no tab stop.
  **Method note from DZ-1b/c:** for *mouse* activation the `HTMLInputElement.prototype.click`
  counter works; for *keyboard* activation it reads 0, because native Enter does not route
  through the DOM `click()` method — use `Page.fileChooserOpened` interception as the signal.
- `playground/scripts/interactions/v-dropzone.mjs:257` already fails from DZ-1b/c
  (`style.display === 'none'`). **Do not edit that file** — the harness is under migration.
  Report its state.

Demo-card changes (`03-click-to-pick` repurposed, `12-folder-drop` as the opt-out example,
`clickIgnore` on `09-css-progress`, prose on the rest) are **DZ-1f**, a separate ticket.
Do not append to `PROGRESS.md` / `TASKS.md`.
