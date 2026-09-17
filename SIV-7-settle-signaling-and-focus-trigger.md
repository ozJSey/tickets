# SIV-7 — settle signaling, `trigger: 'focus'`, and the peerDependencies P1

**Owner, 2026-09-16 (audit walkthrough, stop 11):** all three v-scroll-into-view ideas approved.
This ticket takes settle signaling + `trigger: 'focus'` + the audit P1. **SIV-8 (`keep: true`)
depends on the settle machinery this ticket builds and must run after it** — do not run them
concurrently, they share the executor.

## A. Settle signaling — the wart the README already documents

Today `data-scroll-into-view-state` flips to `idle` before a smooth scroll has moved a pixel, so
"highlight the section when it arrives" — the GitHub-anchor flash, focus-after-scroll, chained
scrolls — has no hook at all.

The machinery is already half-built: `pending-scroll.ts` tracks each scroller's in-flight
destination and forgets it on `scrollend`. Surface it:

- `data-scroll-into-view-state` grows **`travelling`** and a one-shot **`arrived`**.
- Directive option `onSettled(result)`.
- The composable's `scroll()` returns a **promise resolving when every scroller in the chain
  settles**; `result.completed` is `false` when superseded by a newer scroll.

**Keep the native animation.** That is the whole wedge: both incumbents
(`smooth-scroll-into-view-if-needed`, `vue-scrollto`) can only offer callbacks because they replace
the browser's smooth scroll with their own JS animation — and the former has a documented
never-resolving-promise bug (`scroll-into-view-if-needed` #344). Do not regress into that pattern.

Superseded and interrupted scrolls must resolve, not hang — that bug is the named risk here.

## B. `trigger: 'focus'`

```html
<li v-for="opt in options" :key="opt.id" tabindex="0"
    v-scroll-into-view="{ trigger: 'focus', container: '#palette', block: 'center' }">
```

Tab/Shift-Tab focus is never the app's to control — `preventScroll` only helps where you own the
`focus()` call — so the browser's minimal landing wins and your `block`/`offset` are ignored.
Listen for `focusin` and re-issue the directive's scroll one frame later, correcting the landing.

**Be honest about the tie in the README:** CSS `scroll-margin` is already honoured by focus-driven
scrolls and covers the plain "tabbed under a sticky header" case. What it cannot express —
`block: 'center'`, a pinned outer container, offset-as-override — is what this option is for. Say
that plainly rather than implying CSS can't do any of it.

Mirrors the `trigger` option v-select-text already established in this portfolio; keep the naming
and resolution semantics consistent with it.

## C. P1 — `peerDependencies` understates the real Vue floor

`package.json:60` claims `vue ^3.0.0`, but the composable uses Vue 3.2+ APIs, so the **entry breaks
outright on 3.0/3.1** — an install that satisfies the declared range and then fails. Verify the
actual floor by reading the APIs used (do not guess a number), raise the range to match, and note
it in the CHANGELOG as the compatibility correction it is.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **Real-browser checks**: `travelling` observed *during* a smooth scroll and `arrived` exactly
   once on landing; `onSettled` fires after the DOM has actually stopped moving; the promise
   resolves for a superseded scroll with `completed: false`.
2. **Negative control** for A: revert the state-attribute change and watch the arrival check go red
   (proving it is not passing on ambient timing).
3. B: drive a real Tab through CDP into a pinned container, measure the final scroll position
   against `block: 'center'`. Negative control without the option reproduces the browser's minimal
   landing.
4. C: a type/install-level check that the declared range matches the APIs used.
5. The 192-row geometry parity sweep and the 36-row direction sweep must stay green — name them in
   the completion note. Release: **minor**; the major position does not move.
