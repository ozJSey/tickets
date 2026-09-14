# OBS-6 — five defects found by blind certification of `v-observe` 0.2.0 (live on npm)

Found 2026-09-14 by an agent that was given the audit method and forbidden from reading
`tickets/OBS-*`, `tickets/quality/` or `DESIGNS.md`. Everything below was driven in real Chrome via
CDP against **both** `src/` and the built `dist/`, across all 17 cards, all 18 README recipes pasted
and run, and ~30 option pairs.

**`@ozjsey/v-observe` is published at 0.2.0.** There is no unpublished cushion: every defect here is
live for real consumers, and the remedy is a patch release.

**All 241 unit tests are green and blind to all five.** jsdom implements none of the three observers
meaningfully. That is the headline, not a footnote.

## Verdict: partly works

The core is genuinely strong and most of it was proven, not assumed — see "What holds" at the end,
because it matters for judging how much of this package to trust.

---

## 1. README recipe 13 (theme-class watcher) is a silent no-op — **P1, default path**

`<html v-observe="…">` written inside an SFC `<template>` compiles and renders as a nested `<html>`
element *inside* `#app`:

```html
<div><html data-observe-state="intersect:-;resize:-;mutate:idle">…</html></div>
```

The directive binds to that orphan. `document.documentElement` never receives the attribute, and
toggling the real root's class fires **zero** events. No error, no warning.

`README.md:352-366`. The docs gate passes it because it checks that a sample *compiles*, not that it
does anything — and no card demonstrates this recipe, so nothing else caught it.

Documentation-only fix: the recipe has to go through `useObserve`-on-an-element or a mounted hook
against `document.documentElement`, not a template binding. **Then add a card**, or it will rot back.

## 2. `children:added` and `children:removed` are not separable — **P1, default path**

Subscribing to exactly one delivers **both**. README recipe 14 does exactly this
(`on: 'children:added', match: '.card'`).

The un-asked-for direction arrives with its array `undefined`, so a handler written for a
single-type subscription — `e.removed.forEach(…)` — throws `TypeError` **from inside
`flushMutateEvents`**, which aborts the rest of that flush: remaining events dropped,
`mutate:active` never written. Recipe 14 survives only because it happens to type-guard.

Asymmetric with `attr:<name>`, which *is* filtered, and contradicted by README §17's *"Array form
unions the subscription"*.

Cause: `normalizeMutate` collapses both types into one `childList` boolean
(`src/mutate-records.ts:39`); `recordsToEvents` then emits both whenever it is set.

A few lines plus tests. Note the throw-inside-flush blast radius when fixing — a handler that throws
should not be able to drop unrelated events.

## 3. Derived `resize` options are stale until the next geometry change — **P2**

Changing `breakpoints`, `axis`, `on` (mode) or `squareTolerance` produces no handler call and no
`data-observe-state` update until something resizes.

At 400px wide, swapping `{sm:0, md:320, lg:520}` → `{tiny:0, huge:900}` leaves the attribute reading
`resize:md` — **a label that no longer exists in the config**. Switching `on` to `'orientation'`
still leaves `resize:md`, a bracket label in orientation mode. A 1px nudge fixes both.

Contradicts the README's *"Fully reactive"* bullet and *"swaps callbacks live"*. Visible on
playground card 9: slider at 0.40, box 240×200 (ratio 1.20, comfortably inside the band) still reads
`landscape` while the prose directly beneath says it should read `square`.

Contrast `intersect`, where `root` / `rootMargin` / `thresholds` each rebuild and re-report
immediately — so the fix has a correct sibling to copy. `src/resize.ts`, `setupResize` existing
branch.

## 4. `gateOnIntersect` × `debounce` — the two options break each other, and one loses data — **P1**

Both are documented side by side on `ResizeConfig` and `MutateConfig`. `isGated` is checked in the
observer callback, never at flush time, and there is no visible→hidden counterpart to
`onIntersectVisibilityRestored` (`src/gate.ts:34`, called only from `src/intersect.ts:148`).

- A timer started while visible is **never cancelled** when the host goes off-screen: one resize and
  one mutate event fire while `data-observe-state="intersect:hidden"` — precisely the work the gate
  exists to suppress.
- Worse: if the host leaves and re-enters within the debounce window, `resetGateBaseline` clears the
  pending map and re-baselines `lastText`, so **the mutation is lost permanently**. Verified: DOM
  reads `"two"`, no `text` event ever delivered.

Resize recovers (it re-observes). Mutate cannot — a `MutationObserver` cannot re-deliver a past
record. For the README's own live-validation recipe, that is a validator that silently never runs.

The cross-observer gate is this package's differentiator, and *"your mutation was silently
discarded"* is the worst thing a gate can do. Fix this one even if the others wait.

## 5. Playground card 11's prose contradicts the card — **P3**

*"the class toggle also rewrites `style`, so `attr:*` reports both"* — it is the **data-flag** toggle
that rewrites style (`:style="{ borderStyle: flag === 'on' ? … }"`). Driven: under `attr:style`,
"toggle class" produces nothing; under `attr:*` it reports only `attr:class`.
`11-mutate-attr.vue:47`.

## Minor

- Card 9 claims *"Hiding the box reports nothing at all"* but has no hide control
  (`09-resize-orientation.vue:37`) — true, but nothing demonstrates it.
- Four `TEMPLATE_UNDECLARED` advisories: recipes 2, 6 and 14 read `posts` / `items` / `track` /
  `cards` that no `<script>` declares. Pasted as printed, they render nothing for those names.
- `thresholds: [NaN]` reports *"got null"* — `JSON.stringify(NaN)`.
- `v-observe/package-lock.json` was pinned at `0.1.0` while `package.json` says `0.2.0`; npm
  silently rewrote it during the run. The certifier reverted it, but **`npm ci` would trip on it**.

## Did the 0.2.0 repair break anything?

**No** — each 0.2.0 claim was checked explicitly rather than assumed. The `root`/`rootMargin`/
`thresholds` rebuild works; `attr:*` no longer reports `data-observe-state`, with no loop and no
frozen tab even with three observers writing the attribute on one element; the gate and the CSS hook
degrade open *together*, confirmed by deleting each global before boot; a 0×0 box emits no phantom
`square`.

Findings 3 and 4 sit in code the repair touched, but they are **gaps it never closed, not
regressions it introduced**.

## What holds (so the scope of distrust stays honest)

Bare bindings for all three modes; `once`; per-threshold `crossed` up and down at real ratios; all
four scroll directions; a custom `root` proven to be the scroller and not the viewport; `rootMargin`
proven by a slider genuinely rebuilding the observer; breakpoint brackets in object and array form;
per-axis crossings labelled with the bracket *that* crossing entered; orientation with
`squareTolerance`; trailing-edge debounce across a real 7-step mouse drag (zero calls during,
exactly one after); `border`/`content`/`device-pixel` correctly disagreeing; attribute/children/text
mutations including real keystrokes into a `contenteditable` across an Enter-induced node split;
self-removal; the selector filter; the gate going genuinely silent off-screen and returning a fresh
`from: null` in all seven configurations built; fail-open degradation with each observer global
deleted.

`dist/` is genuinely fresh — rebuilt to a temp dir and diffed byte-for-byte. Tarball clean.

## Suggested release

**0.2.1** with findings 1 and 2 at minimum — both are on the path a developer takes straight from
the README, and both are silent. Take 3, 4 and 5 in the same patch if cheap; prioritise 4.

**Do not ship any of it on unit tests alone.** Each fix needs a browser check, because jsdom
demonstrably cannot see any of these.

Reproduction: `run-final.mjs` under the certifier's scratchpad reproduces all five against `dist/`
in one pass.

## One instrument note, already actioned

The certifier hit `pnpm docs` emitting nothing under redirection. Independently confirmed: `docs` is
a **built-in npm/pnpm command**, so `pnpm docs` never ran the gate at all — it exited 0 with no
output. Renamed to `pnpm docs:check` on 2026-09-14; see `playground/README.md`.
