# TT-7 — `flip`: comparative -> fit-based  (v-teleport-to)

**Decided by the owner. Not open for re-litigation.** His words: *"I expect literally for it
to flip."*

**Wrong today** (`src/calculate-position.ts:198-208`): `placement:'bottom'` + `flip` evaluates
`spaceBelow >= spaceAbove ? 'bottom' : 'top'` every tick — re-electing the roomier side, not
holding yours and flipping when it stops working. Measured at 1440x810, 83px popover: flipped
to `top` with **438px unused below** on a 2px difference. Across a 7-position sweep,
`bottom+flip` / `top+flip` / plain `auto` are byte-identical.

**Implement:** (1) stay on the preferred side while the host fits; (2) flip when it cannot
fit and the opposite side can; (3) when neither fits, fall back to comparative so there is
still a defined answer. Both axes. `placement:'auto'` stays vertical-only and comparative.

"Fits" needs a real host height — justify the source in a comment (`getBoundingClientRect` /
`offsetHeight`), state what happens when it cannot be measured (comparative fallback is
honest; treating it as zero is not), and do not add a layout read when `flip` is off.

**Also fix while in this code:**
- `src/constants.ts:9-13` — `BUFFER_TOP` is subtracted from the gap *below* (`:188`) and
  `BUFFER_BOTTOM` from the gap *above* (`:189`). Both names name the wrong edge. Swap.
- **TT-13, and fit-based makes it reachable:** in a 220px boundary `spaceBelow` maxes near
  37px, so `maxHeight` clamps toward 0 — a measured `max-height: 0px` and a 19px padding-only
  popover. There is a dead band where *both* sides are negative and the host renders at zero
  height **with no signal**. "Neither side fits" must become observable, not silent.
- **TT-14 (judge and say):** `TeleportToEventDetail` reports `availableSpace` for the chosen
  side only, so a consumer cannot show *why* without re-measuring and re-hardcoding the
  buffers — which `02-placement-flip.vue` now does. Add the losing side + a neither-fits
  signal, or declare it out of scope. Do not half-do it.

**Tests:** the existing ~25 flip tests (`vTeleportTo.test.ts` `describe('flip option')` ~`:629`,
horizontal ~`:1882`) **pass under either semantics** — they are not evidence. Audit each.
New tests must pin: preferred side fits with less room -> stays (the old code fails this);
cannot fit, opposite can -> flips; neither fits -> comparative fallback and observable;
same four horizontally; `auto`+flip unchanged; `flip:false` never moves; unmeasurable height
degrades to the documented fallback.

**Acceptance:** `npm test` green across 5 projects (baseline **646** after T0);
`npx tsc --noEmit`, `npm run build`, `npm pack --dry-run` clean; from `playground/`,
`pnpm typecheck && pnpm smoke && pnpm smoke:dist` clean. README `:82`, `:242-243` and
`types.ts` jsdoc updated. Update **only the explanatory prose** in
`playground/src/demos/v-teleport-to/02-placement-flip.vue` — another agent rebuilt that card
with a bounded 420x360 `boundary`, a scrollable canvas and live space chips, browser-verified.
Do not restructure it.

**Browser-verify** (owner rule: *better to fail a test than claim something we don't do*):
`pnpm dev`, drive Chrome via `playground/scripts/lib/cdp.mjs`, read `data-teleport-placement`
back and prove: (a) `bottom`+flip now stays where the old code flipped, popover fitting;
(b) it does flip when it genuinely cannot fit; (c) `right`->`left` likewise inside the bounded
stage; (d) the neither-fits state is reachable and observable. Scratchpad only, never the repo.

Do not append to `PROGRESS.md` / `TASKS.md` — the coordinator owns those.
