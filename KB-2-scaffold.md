# KB-2 — scaffold `@ozjsey/v-keyboard-navigation`

Verdict and full survey: `DESIGNS.md` → KB-1. Standards: `tickets/_STANDARDS.md`.
**Name settled by the owner**: directive-based is confirmed, so `v-keyboard-navigation` it is.

## Why it exists — and the wedge is measured, not assumed

Roving-tabindex libraries do **not** call `scrollIntoView`, deliberately: the APG says the benefit
of roving tabindex over `aria-activedescendant` is that *"the user agent will scroll the newly
focused element into view."* Radix, Reka, Primer, keyux and makeup all contain zero occurrences of
it. **But the UA scrolls badly.** Measured in Chrome, 200px container, 40px items, one ArrowDown
per step:

```
native focus()                      0,0,0,0,0,120,120,120,240,240,240,360   ← centres: 3-item lurch
preventScroll + block:'nearest'     0,0,0,0,0, 40, 80,120,160,200,240,280   ← 1-item follow
```

Two facts that follow, both verified:
- **`focus()` then `scrollIntoView({block:'nearest'})` is a silent no-op** — the UA centres first,
  so `nearest` finds the item already visible. `preventScroll: true` is **mandatory**. Reka UI
  2.10.4 hits exactly this path, so its stated `nearest` intent does not take effect in Chromium.
- **`smooth` is the wrong default here** — >400ms to settle versus ~30ms key repeat, so the list
  visibly lags the focus ring. Use instant.

Clears the bar `v-trap-focus` failed, and it is the inverse case: VueUse has **nothing** (all 248
exports of `@vueuse/core` 14.4.0 enumerated — `useFocus`, `onKeyStroke`, `useMagicKeys` are
primitives, no composite-widget helper). The Vue-native field is one 2-year-stale directive at
3,342 weekly downloads and one at 66.

## Scope

**In (v1):** 1D composite widgets — roving tabindex, arrow keys, Home/End, typeahead,
wrap-vs-clamp per role, orientation, skipping disabled/hidden, last-focused memory, **controlled
scroll**, focus state as a `data-*` attribute, `aria-activedescendant` as an opt-in mode.

**Worth doing because nobody does it:** PageUp/PageDown as a **real visible page**. Radix and Reka
both map them to first/last, which is not what the APG describes.

**Out:** app-level shortcuts — **do not build**, VueUse `onKeyStroke`/`useMagicKeys` own it outright.
Focus-on-route-change — different concern. **2D grid — deferred to v2**; the platform deferred it
too, and a half-done grid is the worst possible failure mode.

## Adopt the platform's vocabulary

`focusgroup` is a real HTML attribute (Chrome 150, behind a flag; Firefox positive-pending). Use its
words — `toolbar | tablist | menu | menubar | listbox | radiogroup`, `inline | block`,
`wrap | nowrap`, `nomemory` — so this is a forward-polyfill whose migration path is *"delete the
directive, add the attribute, keep the scroll option."* Its permanent non-goals (no typeahead, no
grid in V1, no selection management, scrolling left to the UA) are exactly what this package adds.

## The bare binding, zero options (`_STANDARDS.md` B7)

```vue
<ul v-keyboard-navigation>
  <li><button>Bold</button></li>
  <li><button>Italic</button></li>
</ul>
```
Finds focusable descendants; makes exactly one tabbable (the first, or whichever carries
`aria-current` / `aria-selected="true"`); arrows on both axes unless `role`/`aria-orientation`
narrows it; Home/End; typeahead on printable characters; wrap default per role (clamp for
toolbar/listbox, wrap for menu/menubar/tablist/radiogroup); `focus({preventScroll:true})` +
`scrollIntoView({block:'nearest'})`; remembers last-focused across Tab out/in; skips `[disabled]`,
`[aria-disabled="true"]`, `[hidden]`; suppresses typeahead inside text fields and contenteditable.

## Where it must refuse to guess

- **Never inject a container role.** A container that gains `role="listbox"` without
  `aria-selected` on every option is *worse than plain markup* — the concrete form of
  "half-correct is worse than none."
- **Never write selection state** (`aria-selected` / `aria-checked`). The app's model owns it.
- **Never infer orientation from measured layout** — it would change behaviour on resize. Read
  `role` / `aria-orientation`, else bind both axes.
- **DOM order, never CSS `order`/grid placement.** Document it.

## Non-negotiables — the failure mode is silent to the developer who ships it

1. **Exactly one tabbable item, always** — maintained across `v-for` add/remove/reorder, `v-if`,
   and async loads. A group that reaches zero tabbable items **disappears from the keyboard
   entirely.** Needs a MutationObserver and tests that mutate the list mid-flight. This is the #1 bug.
2. **Never intercept keys inside text inputs, `select`, or contenteditable.**
3. **Never `preventDefault` a key you did not handle.**
4. **Screen-reader browse mode:** in NVDA/JAWS virtual cursor, arrows never reach the handler.
   **Keyboard testing alone cannot validate this package.**

## Scroll: carry your own

~40 lines. Do **not** depend on `v-scroll-into-view` — packages here are independent, and the two
genuinely differ (that one is declarative and edge-driven with `smooth`; this needs imperative
per-keystroke steps with `preventScroll`, where `smooth` is wrong). **Borrow its option shape
verbatim** — `container`, `offset: {top,left}`, `behavior`, `block` — so learning one teaches the other.

## Deliverables (all of `_STANDARDS.md`, not a subset)

- `src/` split into single-purpose modules behind a thin barrel, plus `ARCHITECTURE.md` naming the
  invariant. Suggested: `items.ts` (collect + filter focusable), `roving.ts` (the one-tabbable
  invariant + MutationObserver), `keys.ts` (key → intent), `typeahead.ts`, `scroll.ts`,
  `directive.ts`, `types.ts`, `index.ts`.
- Vue-first, typed, no `@ts-ignore`, no casts.
- **Playground tab in this run, not a follow-up** — one card per feature: toolbar, tablist, menu,
  listbox with a long scrolling list (the wedge card — put 200 items in a 200px box and show the
  one-item follow), radiogroup, typeahead, wrap-vs-clamp, dynamic list mutation, the
  `aria-activedescendant` mode, and **a card built for manual screen-reader verification** with a
  documented walkthrough of what to listen for per pattern.
- README: the APG patterns committed to in v1 (Toolbar, Tabs, Menu/Menubar, Listbox single-select,
  Radio Group) and those **documented as unimplemented** (Grid, Treeview, Combobox, Feed, Carousel).
  State plainly: *this implements the keyboard interaction of these patterns; it does not make your
  markup a listbox — roles, names and selection remain yours.*
- `CHANGELOG.md`. Version **0.1.0** — unaudited packages stay 0.x (`_STANDARDS.md` B9).
- TDD, and **mutation-test the suite**; four agents this week found their own tests weaker than
  they looked.
- Browser-verified with real key events (`Input.dispatchKeyEvent`), not synthetic dispatches.
  The scroll wedge must be proven by reading `scrollTop` back per keystroke, as measured above.

Do not publish. Do not append to `PROGRESS.md` / `TASKS.md`.
