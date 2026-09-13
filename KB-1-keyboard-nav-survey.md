# KB-1 — `v-keyboard-navigation`: uniqueness survey, then scope

**Read `tickets/_STANDARDS.md` first.** Read-only research; produce a verdict, change nothing.

**Why this survey gates the build.** This portfolio enforces its uniqueness bar by
cancellation (`CLAUDE.md`, `TASKS.md` Decisions). `v-trap-focus` had 150 passing tests, a
plugin, a composable, a nested stack and 11 options and was **archived anyway** on 2026-08-09
because `focus-trap`, `focus-trap-vue` and VueUse's `useFocusTrap` already served the niche.
Keyboard navigation is a more crowded niche than focus trapping. **"Do not build this" is a
valid, valuable verdict.**

## Owner requirements — the package is not just key handling

> "keyboard navigation MUST support scrollIntoView and focus out of the box, focusWithin etc,
> classic keyboard navigation related functionalities."

So it owns the whole focus-movement story:
- move focus with the arrows (roving tabindex **or** `aria-activedescendant` — decide which,
  or both, and say why)
- **scroll the newly-focused item into view.** An APG requirement for any listbox longer than
  its viewport, and the part most roving-tabindex libraries leave to the consumer. **Test the
  wedge here specifically** — check whether VueUse, Reka UI, Headless UI, Ark/Zag, Tabster,
  Primer or any standalone lib does this out of the box.
- focus state surfaced for styling (`:focus-within` / focus-visible)
- the classic set: Home/End, typeahead, wrap vs clamp, orientation, Page Up/Down, 2D grid

## Survey properly — real numbers and real docs, not recall
VueUse (`onKeyStroke`, `useMagicKeys`, `useFocus`, `useFocusWithin`); Headless UI Vue; Reka UI
(ex Radix Vue); Ark UI / Zag.js; PrimeVue; Vuetify; Microsoft **Tabster**; GitHub Primer +
`@github/*` elements; React `roving-ux` / `@radix-ui/react-roving-focus`; npm searches for
`roving-tabindex`, `keyboard-navigation`, `arrow-key-navigation`, `focus-manager`, `vue-a11y-*`.
For each: version, last publish, weekly downloads, Vue directive or not, and **which of the
requirements above it covers**. The WAI-ARIA APG composite-widget patterns define "correct".

## Answer, in order
1. **Is there a wedge?** State it as a README lead, or say there is none. Candidate to test
   hardest: *a directive that upgrades markup you already wrote*, versus component libraries
   that require restructuring your app into their components — combined with scroll-into-view
   and focus state as one thing. Is that combination genuinely unserved?
2. **Does it clear the bar `v-trap-focus` failed?** Argue explicitly against that precedent.
   If the honest answer is "this is v-trap-focus again", put that in the first sentence.
3. **Scope.** Roving list/menu; 2D grid; app-level shortcuts (VueUse already does this);
   focus-on-route-change. Which are one package, which are separate? Recommend one.
4. **The `v-scroll-into-view` tension.** Scrolling the focused item into view is already this
   portfolio's `v-scroll-into-view`, but packages here are independent with **no
   cross-dependencies**. So this package carries its own scroll logic, or the two compose
   without a hard dep. Recommend, and say what that means for both packages.
5. **DX and smart defaults.** Per `_STANDARDS.md`, the bare binding must be correct with no
   options. What does `v-keyboard-navigation` on a bare `<ul>` do, with nothing configured?
   What does it infer (orientation from layout? items from focusable children? role from the
   host?) and where must it refuse to guess?
6. **A11Y-1 relevance.** The board carries a rule — *a directive that makes an element
   clickable must make it keyboard-operable, preferring a real focusable control over injected
   ARIA*. Say plainly whether this package relates to that gap or is a red herring; packages
   are independent, so a shared package cannot be the fix.
7. **Name.** All free as of 2026-09-05: `v-keyboard-navigation`, `v-keyboard-nav`, `v-roving`,
   `v-roving-tabindex`, `v-arrow-nav`, `v-keynav`, `v-navigate`. Recommend one matching the wedge.
8. **The honest downside.** A half-correct roving tabindex is *worse than none* — it silently
   breaks screen-reader users while looking fine to a sighted developer. What must it get right,
   and is that scope realistic? Any APG pattern not fully implemented must be documented as
   unimplemented, never fudged.

**Return:** verdict up front (build / build-narrower / do not build); incumbent table with real
numbers; the wedge as a README lead; recommended scope and name; risks; and only if "build",
a Vue-first API sketch plus the APG patterns it would commit to. Cite what you checked.
