# PG-25 — should the playground become VitePress?

Owner, 2026-09-15: *"Our playground can also use vitepress for cleaner execution."*

Recommendation up front: **yes for the docs site, no for the demo harness — and the two can be the
same site.** The version that fails is a wholesale migration, for one specific reason below.

## What VitePress genuinely buys

DOCS-1 and DOCS-2 ask for a documentation view per package, a changelog view, and a Pages deploy.
Hand-rolling that is exactly what `scripts/docs.mjs` (21 KB) and `scripts/standalone-pages.mjs`
already are. VitePress gives all of it for free: Markdown-driven pages, sidebar nav, client-side
search, dark mode, and a first-class GitHub Pages build. Twelve READMEs and twelve CHANGELOGs
become twelve routes with no bespoke code. That is a real reduction in surface we maintain.

## What a wholesale migration would break, and why it is not negotiable

The verification harness is coupled to the playground's DOM contract, not to its styling:

- `scripts/interactions.mjs` + 10 spec files find every element through
  `section[id="demo-<file>"] .demo__stage`. That scoping is load-bearing — PG-4/5 exist because a
  section-wide `querySelector('button')` reached the *card's* chrome and passed for the wrong
  reason.
- `scripts/geometry.mjs` measures rendered layout across slider positions.
- `scripts/smoke.mjs` asserts every card mounted.
- In-browser "Edit code" recompiles the SFC as you type, via `vue3-sfc-loader`.

Move the demos into Markdown and that contract changes shape, which quietly invalidates **330
interaction checks and every geometry sweep** — the only evidence in this repo that the libraries
work, since jsdom has no layout. Trading measured behaviour for a nicer site is a bad trade at any
exchange rate.

## The version that works

VitePress renders Vue components inside Markdown natively. So:

1. VitePress owns the **site**: nav, routing, search, theme, Pages deploy, docs + changelog pages
   generated from each package's `README.md` / `CHANGELOG.md`.
2. `DemoCard.vue` and the demo SFCs are **imported into the Markdown pages unchanged**, so
   `section[id="demo-*"]` and `.demo__stage` survive byte-for-byte and every runner keeps working.
3. The runners point at VitePress's dev server instead of the current Vite one. `scripts/lib/port.mjs`
   and `boot.mjs` already abstract that; `waitForBoot` needs a new readiness signal, which is
   PG-11c's `data-demo-state` attribute — **already on the board as the highest-leverage change.**

## Do it in this order, or not at all

1. PG-11c first (`data-demo-state`), so readiness does not depend on the current DOM.
2. Stand VitePress up **beside** the existing app, one tab ported, and prove `interactions` is green
   against both.
3. Only then port the rest.

If step 2 cannot get a tab green, stop — the answer is "VitePress for docs, Vite app for demos,
two builds", which is still better than today.

## Cost note

This lands on top of CI-1, DOCS-1 and DOCS-2, all of which touch the same deploy. Sequencing them
as one piece of work is cheaper than three passes over the same Pages config.
