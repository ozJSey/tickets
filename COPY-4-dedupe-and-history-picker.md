# COPY-4 — `v-copy`: `dedupe` option + the flagship history-picker card

Full design and reasoning: `DESIGNS.md` → "COPY-1 / DEMO-1". Read it first.
Standards: `tickets/_STANDARDS.md`. Standing test rule: `BOARD.md`.

**Why this ticket matters more than its size suggests.** The owner: *"the entire reason of why
I created v-copy is to show it in following demo: we copy, then by using v-teleport-to we have a
dropdown to open, on dropdown click user can decide what to copy again."* The history stack is
not a feature of this package, it is the point, and the picker has never existed. A registry
survey found **no Vue or browser clipboard library ships history at all** — `keywords:copy-history`
returns zero packages — so this is also the package's strongest claim to exist.

## Part A — the `dedupe` option (library)

```ts
export type DedupeCompare = 'exact' | 'trim' | 'loose'
export interface DedupeConfig {
  compare?: DedupeCompare | ((a: string, b: string) => boolean)
}
// CopyConfig + CopyPluginOptions gain:
dedupe?: boolean | DedupeConfig   // default TRUE
```

Mirrors the existing `feedback?: boolean | FeedbackConfig` / `announce?: boolean | string` shape,
so it inherits a resolution path readers already know. `Resolved` carries a **pre-resolved
predicate** (`((a,b)=>boolean) | null`) so `history.ts` stays branch-free. Note `resolveBinding`'s
`v === false` early-return enumerates every field — TypeScript will force `dedupe: null` there.

**Dedupe on write, not on read.** The bound `ref` belongs to the consumer (README: *"The `ref` is
yours"*), so there is nowhere to put a derived view for the plain binding form; `max` must count
what the user sees; and read-time dedupe breaks the pinned in-place-mutation invariant
(`vCopy.test.ts:228`).

**Promote, do not ignore.** On a hit, remove **every** prior match (iterate backwards so splices
do not skip) and `unshift` a fresh entry — in rich mode with a fresh `at`, or the list sorts by
recency while showing a stale timestamp. Removing every match makes the option idempotent, which
the demo's toggle depends on.

**Order inside `record()`:** plain-mode failure early-return (unchanged) → build entry →
`if (r.dedupe && result.success)` splice matches → `unshift` → cap → update `controllerObj.last`.

**`max` pinned:** dedupe first, then the cap; a promotion never frees or consumes a slot.
`max:5`, `[e,d,c,b,a]`, re-copy `c` → `[c,e,d,b,a]` (dedupe) vs `[c,e,d,c,b]` with `a` evicted
(no dedupe). **That divergence is the demo's payoff — make it visible.**

**Compare `'exact'` by default**, on `CopyResult.text`. A history must return exactly what was
copied; collapsing `"foo "` into `"foo"` makes one payload unreachable. Comparison never mutates
the stored entry. Failed copies never merge and never evict. Mixed sinks are real (two bindings,
only one `.rich`), so extract text mode-agnostically.

**Default on** — verified zero compat cost: the `v-copy` on npm is egoist's 0.1.0 from 2017, a
different package. Budget: whole feature ~25 lines; ESM gzip must stay under 3.0 KB (2.81 today).

## Part B — `DemoMeta.uses` (playground registry)

Add `uses?: string[]` to `DemoMeta` in `playground/src/registry.ts` — it flows through
`buildLibrary` with no other change. Three consumers: a distinct chip in `DemoCard.vue`; the
filter haystack in `App.vue`; and **the editor hint fix** — `DemoCard.vue` currently hardcodes
`Imports resolve to the library sources in ../{{ demo.id.split('/')[0] }}/`, which is simply
false on any cross-library card.

Write the placement rule into `instructions/playground.md`: *a cross-library card lives in the tab
of the library whose feature it proves; every other package it touches goes in `uses`.*

## Part C — the card: `playground/src/demos/v-copy/13-history-picker.vue`

Spec is in `DESIGNS.md`. Load-bearing details:

- `reactive<CopyController>({ sink: [], rich: true, dedupe: true, max: 6 })`. Pre-seed `sink: []`
  so `picker.history` is an array on first render.
- A separate `rawLog` of every attempt, collected by **one `@copy-result` listener on a wrapper
  div**. This works because `v-teleport-to` does not move the host in the DOM — it only writes
  `position: fixed` coordinates — so the dropdown is still a descendant and the event bubbles.
  That is a real, non-obvious difference from `<Teleport>`; say it in the prose.
- Controls: `dedupe` checkbox, `max` number input, Clear. **The template must read
  `picker.dedupe`** — that read is what re-renders and re-resolves the option. A knob held in a
  variable the template never reads silently never takes effect. Comment it.
- Trigger inside an `overflow: hidden` clipper, so the escape is visible.
- Dropdown rows carry **their own `v-copy` binding**, not `picker.copy(e.text)` — `ctrl.copy()`
  runs on the last-mounted driver, so the `[data-copied]` flash would land on an unrelated chip.
- The dropdown **stays open after a pick**, so the picked row visibly jumps to index 0.
- Counters showing `copies made` vs `history length / max` — the dedupe divergence, on screen.
- A paste target `<textarea>` so a human can verify by hand.

Manifest entry per `DESIGNS.md`, with `uses: ['v-teleport-to']`. Update counts: `playground/README.md`
v-copy row 12→13, `instructions/playground.md` 83→84 demos.

## Acceptance

- TDD throughout. All 47 existing `v-copy` tests green plus new ones covering every pinned
  semantic above (promotion, all-matches removal, max interaction both ways, mixed sinks, failure
  handling, the three compare modes, plugin default and per-binding override).
- `npm run build`, `npm pack --dry-run` clean; gzip under 3.0 KB.
- From `playground/`: `pnpm typecheck`, `pnpm smoke`, `pnpm smoke:dist` green.
- **Browser-verified** (owner rule — jsdom has no clipboard): drive the card over CDP via
  `playground/scripts/lib/cdp.mjs`, and prove the seven-step script in `DESIGNS.md` §2.4 —
  especially that re-copying a row puts that text on the real clipboard (`readText()` after
  `Browser.grantPermissions` for `clipboardReadWrite`), that the row moves to index 0, and that
  dedupe on/off makes `rawLog` and `picker.history` diverge. Scratchpad only, never the repo.
- Do **not** add permanent checks to `playground/scripts/interactions/` — that harness is under
  active migration. Report what the checks should be; another ticket lands them.

**Coordination:** agents may be active in `v-teleport-to`, `v-dropzone` and `playground/scripts/`.
Stay in `v-copy/`, `playground/src/`, and the two doc files named above. You *use* `v-teleport-to`
in the card; do not edit it.
