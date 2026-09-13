# FIT-1 — `v-fit-children` audit findings (in the published package)

Verdict: **WORKS.** 2,193 measurements, zero half-clipped children, worst overshoot +0.5px inside
its own EPSILON, monotonic everywhere. Driven against the **published** `@ozjsey/v-fit-children@2.2.0`
installed from the registry into a clean Vite + Vue app. Supply chain clean: `dist` byte-identical
to a fresh rebuild **and** to the npm tarball.

These are the defects the method still found. **Real users can hit F1 today.**

**F1 — P1, default path, ships in 2.2.0.** No `fit-children-updated` event fires when the first
pass hides *every* child. `state.visible` starts as `new Set()` (`src/directive.ts:76`); if the
first computed fit is also empty, `sameRun` is true and `data === lastDispatchedData` (both
`undefined`), so `src/visibility.ts:70-76` skips the dispatch.

Driven: bare binding, 9 chips into a 90px container. All 9 hidden, `data-v-fit-state="overflowing"`
correct, **event never dispatched** — at mount, after a `v-if` remount, and after a resize that
keeps the run empty (90→91px, still silent). Recovers only once one child becomes visible.

**Consumer symptom: an over-full container with no "+N more" badge, at exactly the widths where the
badge matters most.** CSS-only styling still works; every event-driven consumer does not.
**Warrants a 2.2.1.**

**F2 — P2, and it is a test passing for the wrong reason.** `isOverflowing` in the event goes stale
whenever the visible set does not change. The attribute write happens *before* the early return, so
`data-v-fit-state` is always right and the event is not.

Driven: 5 chips all carrying `data-v-fit-keep`, swept 140→700px. The attribute flips
`fits`↔`overflowing` correctly at every width; the event's `isOverflowing` stays `false` throughout.

**The README changelog lists this exact case as fixed**, and `vFitChildren.test.ts:262` covers it —
but only on the **mount** pass, where `state.visible` is empty so the dispatch always happens. The
resize path is untested and broken.

**F3 — P2.** The two documented ways to pin a child disagree about `data`. `isDataChild`
(`src/dom.ts:79-84`) excludes `keepVisibleEl`-matched children from the data index but **not**
`data-v-fit-keep` ones:

| width | actually hidden | `hiddenData` with `data-v-fit-keep` | with `keepVisibleEl` |
|---|---|---|---|
| 190 | E | **`[]`** | `["E"]` |
| 160 | D, E | **`["E"]`** | `["D","E"]` |

`hiddenIndices` is correct in both. Combining README "Data mapping" with "Keeping elements visible →
Option B" silently drops entries from the "+N more" list.

**F4 — cosmetic.** Playground card 08 renders a trailing separator with nothing after it
(`Ada · Grace · Alan · Edsger ·` + `+2 more`) in the canonical demo for the feature.

**F5 — docs describe a package that does not exist.** `FitChildrenPlugin`, `DIRECTIVE_NAME` and
`data-fit-children-state` appear **nowhere** in `src/`, `dist/`, the README or the published
tarball — only in `CLAUDE.md:112` and `instructions/v-fit-children.md:15-28`. The brief also still
describes the rAF/ghost/host-sizing engine that `src/` explicitly removed, and claims "2.1.0 on npm;
3.0.0 pending publish" (npm has 2.2.0). `playground/.../manifest.ts:7` prints the same wrong text in
the UI. **The README is the accurate document and matches the published artifact exactly.**

**F6 — README recipes do not compile as pasted.** Quick start, keepVisibleEl Options A and B, and
the inline-badge recipe all `v-for="tag in tags"` with `tags` never declared; the badge snippet also
uses undefined `hiddenCount`/`onUpdate`. All four behave correctly once `tags` is supplied. "Data
mapping" is the only self-contained recipe.

**F9 — housekeeping.** A stray `ozjsey-v-fit-children-3.0.0-rc.4.tgz` sits in the package root,
*ahead* of what is published and built from a different (single-format, no-CJS) config.

## Acceptance

- F1 and F2 fixed with **resize-path** regressions — both bugs live in the path the existing tests
  do not exercise. F2's existing test must be shown to fail against the resize case before the fix.
- F3: one answer for both pin mechanisms, whichever is correct, and the README says which.
- F5: `CLAUDE.md` and `instructions/v-fit-children.md` corrected against the source. **Do not
  "restore" the missing exports — they were removed deliberately; the docs are wrong, not the code.**
- Confirmed working and not to be regressed: gap as a floor, `widthRestrictingContainer`,
  trailing-margin accounting, `v-show` composition, real keystrokes into card 03, card 02's
  self-resizing trigger with no oscillation over 20 samples, unmount restoring every child, a single
  injected `<style>` across 6 mount cycles, and **188 rAF-sampled frames during a hard resize sweep
  with zero clipped frames.**
