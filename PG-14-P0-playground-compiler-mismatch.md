# PG-14 — **P0.** The playground compiles with Vue 3.4.15 and runs Vue 3.5.41

Found by the independent `v-copy` audit, 2026-09-06. **This invalidates a class of verification
across the whole portfolio, not one library.**

## The mechanism

`playground/src/components/DemoCard.vue` always routes demos through `compileSfc`
(`src/sfc-runtime.ts`) — the registry globs sources `?raw`, so `@vitejs/plugin-vue` **never**
compiles a demo. `vue3-sfc-loader` bundles `compiler-sfc` **3.4.15**, while the app runs Vue
**3.5.41**.

3.4.15 emits a directive host inside `v-for` as `createElementVNode("li", {key}, …)` with
**patchFlag 0**, inside a `STABLE_FRAGMENT`. It is never collected into `dynamicChildren`, so it is
never re-patched. 3.5.41 emits `withDirectives((openBlock(), createElementBlock("li", …)))` for the
identical template.

**Consequence: a directive inside a `v-for` never receives `updated`.** The auditor instrumented the
live directive via `app._context.directives.copy` and logged **zero** `updated` calls for a card's
`<li>`s, while non-`v-for` cards logged them normally. A side-by-side repro in a real Vite + Vue
3.5.41 app confirms the same code works there.

## Why this is a P0

**Every directive in this portfolio is configured through its binding value.** If `updated` never
fires, no option change reaches the directive. So:

- **Option reactivity cannot be verified in any `v-for` card, for any of the seven libraries.**
- Three `v-copy` cards are dead *because of the playground*, not the library: card 13's `dedupe` /
  `compare` / `max` controls are inert for the six chips (uncheck dedupe, copy the same chip three
  times → `copies made 3`, `history 1 / 6`; `max` 2 with 6 entries renders a self-contradicting
  "history 6 / 2"); card 11's `max` slider does nothing; card 03's **Clear** button permanently
  orphans the sink — the ref holds a fresh array while the directive keeps writing into the old one.
- **Card 13 is the card COPY-4 signed off with 19 browser checks.** Half of it is live and half is
  inert; the checks happened to exercise the live half. That is the sharpest available evidence that
  "browser-verified in the playground" has been worth less than it appeared all session.

## What to fix

Make the compiler match the runtime. Options, to be weighed and one recommended:

1. **Upgrade `vue3-sfc-loader`** to a build whose bundled `compiler-sfc` matches the app's Vue, and
   pin the pair so they cannot drift again.
2. **Supply the compiler explicitly** — `vue3-sfc-loader` accepts injected modules; hand it the
   app's own `@vue/compiler-sfc` rather than its bundled copy.
3. **Assert the versions at boot** and fail loudly on mismatch, regardless of which fix lands. A
   silent divergence between compiler and runtime is the defect class; a version check is the guard.

Whichever lands, **(3) is not optional** — this bug was invisible for the life of the playground.

## Acceptance

- A `v-for`-hosted directive receives `updated` when its binding changes. Prove it by instrumenting
  the live directive and counting calls, the way the audit did — not by observing that a card
  "looks right".
- The three broken `v-copy` cards behave as their own prose promises: dedupe off makes "copies made"
  and "history" diverge; `max` evicts live; **Clear** does not orphan the sink.
- A boot-time assertion fails the app if `compiler-sfc` and the runtime disagree.
- **Re-run every existing interaction check afterwards** — `v-dropzone` (51) and `v-select-text`
  (23). Any check that was passing against an inert control is a false green and must be re-judged.
- Add this to `AUDIT-1`'s method: when a card's control appears inert, distinguish *library bug*
  from *playground bug* before filing.

## Standing implication for the board

Until this is fixed, **no claim of the form "verified in the playground" may be made about an option
whose control lives inside a `v-for`.** That includes work already reported as done this session.
