# WBC-3 — `vue-write-behind`: playground tab + documentation view

**This should have shipped with WBC-2.** `tickets/_STANDARDS.md` item 4 — taken from the owner's
own words — says a new package is *"registered in the playground, one card per feature, added in
the same run as the code — not as a follow-up."* WBC-2 was scoped as "phase 1: the package" and
deferred this, which broke the rule. The owner noticed: *"I also don't see new libraries in the
playground."*

The package exists at `vue-write-behind/` with 193 tests across Vue 3.5 / 3.3 / SSR, 25 mutations
none surviving, and a `playground.html` proving the no-jump claim. Design: `DESIGNS.md` → WBC-1.

## Wiring (all four together, or the tab breaks)

`playground/vite.config.ts` (alias), `playground/tsconfig.json` (paths),
`playground/src/libraries.ts` (import + registry map + install line), and a new
`playground/src/demos/vue-write-behind/manifest.ts`. Note this is the first **non-directive**
package to get a tab — the registry does not require a directive, but check `libraries.ts` and
`DemoCard.vue` for anything that assumes one.

## Cards — one per feature

**The flagship card, and the only one that proves the product: the cell must not jump.** A grid of
inputs against a deliberately slow server that echoes the value transformed (uppercased). Type
continuously while a request is in flight; the field must keep exactly what was typed and never
flicker to the server's answer. WBC-2 proved this over CDP with real keystrokes — 11 keystrokes,
2 PUTs, no jump — and mutated the demo to confirm the check goes red. **Port that into a card a
human can see doing it.**

Then: coalescing (a keystroke counter vs a request counter, diverging live); failure and retry
(a fail-then-recover endpoint, with the key staying dirty and going out again carrying the *newest*
value); `pending` / `inFlight` / `failed` surfaced as live state; batch mode vs per-key
`allSettled`; `discard()` as the only way to lose a write; flush-on-tab-hide.

The dev server already fakes `/api/upload*` in `vite.config.ts` — a similar mock is needed here.
**Note DOCS-1: those routes are a `configureServer` middleware and do not exist in a static build,
so anything added the same way will 404 on GitHub Pages.** Coordinate, or use a shape that survives.

## Documentation view

Per DOCS-1, the docs view renders the package README rather than being authored twice. If DOCS-1
has not landed, build the tab so its docs slot is ready rather than inventing a second mechanism.

## Acceptance

- `pnpm typecheck`, `pnpm smoke`, `pnpm smoke:dist` green; playground demo count updated in
  `playground/README.md` and `instructions/playground.md`.
- The no-jump card is verified in a real browser with **real typed key events**, not synthetic
  `input` dispatches, and with a negative control proving the check can fail.
- Nothing in the tab claims behaviour listed as UNPROVEN in WBC-2: real `visibilitychange` on an
  actual tab hide, `keepalive` delivery, and real offline.
