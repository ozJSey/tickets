# KB-3 — skipping must be opt-in and independent of `disabled`

Owner, 2026-09-13:
> "vKeyboard navigation should allow skipping, like no point of navigating to disable IF they CHOOSE
> to (could be opt in with attribute, it doesn't always mean disabled)"

Two separate requests, and a third problem found while confirming them.

## 1. A "skip me" attribute that does not mean disabled

`src/items.ts` currently derives skippability **entirely** from disabled/hidden state:
`[disabled]`, `[aria-disabled="true"]`, `fieldset[disabled]`, `[hidden]`, `[inert]`,
`[aria-hidden="true"]`, inline `display:none` / `visibility:hidden`.

There is no way to say *"this is a perfectly live control, just do not stop on it with the arrows."*
Real cases: a group label or section header that happens to be focusable, a separator, a loading
placeholder, a "load more" row that should only be reachable by Tab, a decorative item.

**Name it against the vocabulary the package already adopted.** KB-2 deliberately took the HTML
`focusgroup` attribute's words (`toolbar | tablist | menu | menubar | listbox | radiogroup`,
`inline | block`, `wrap | nowrap`, `nomemory`) so the package is a forward-polyfill. Check what the
`focusgroup` explainer and the CSSWG/OpenUI discussions call an opted-out descendant before
inventing `data-kbd-skip` — if the platform has a word, use it.

## 2. Skipping disabled items must become a choice, not a default

It is unconditional today. The owner wants it opt-in. **The APG agrees with him, and in the
direction that matters:** for **menus and menubars** disabled items should generally remain
focusable, so the user learns the option exists. Skipping them hides information. For **toolbars**
and **listboxes** skipping is more often correct.

So there is no single right default — it is **per pattern**. The package already reads `role` /
`aria-orientation` to pick wrap-vs-clamp per role; the same mechanism should pick the
skip-disabled default per role, with an explicit option overriding it.

Decide and document:
- the per-role defaults, cited to the APG pattern, not guessed
- the option name and shape (`skipDisabled?: boolean`, or something that also covers `aria-disabled`
  separately — note `disabled` removes an element from the a11y tree while `aria-disabled` keeps it
  announced, which is precisely why the APG treats them differently)
- whether `hidden` / `inert` / `display:none` stay unconditional. **They probably should** — an
  element with no box or `inert` cannot be focused at all, so "skipping" it is not a policy choice.

## 3. The asymmetry that makes this dangerous either way

**A skipped item is invisible to arrow navigation but still in the accessibility tree.** A
screen-reader user browsing by role reaches it, then cannot get to it with the arrows. Sighted
keyboard users and screen-reader users navigate two different lists.

Whatever is built must address this rather than inherit it:
- a skipped-but-present item should be `aria-disabled` or otherwise *explained*, not silently
  unreachable
- the README must state the consequence plainly, because a developer adding a skip attribute will
  not think about browse mode
- **this is the part no automated check can confirm.** The package already ships its screen-reader
  claims marked UNPROVEN with a manual walkthrough card (card 13). Extend that card rather than
  asserting anything new.

## Acceptance

- TDD; baseline is 99 test declarations and 39/39 mutants killed — **keep mutation testing**, it has
  caught a real test gap in this package before.
- Tests for: the skip attribute on a live control; skipping disabled on and off per role; the
  default matching the APG for each committed pattern (Toolbar, Tabs, Menu/Menubar, Listbox, Radio
  Group); a group where **every** item is skipped (it must not become unreachable by Tab — the
  one-tabbable invariant is this package's #1 bug class and a fully-skipped group is the obvious way
  to break it); and typeahead, which must agree with arrow navigation about what is skippable.
- A playground card, and an extension to the manual screen-reader walkthrough.
- **Browser-verified with real `Input.dispatchKeyEvent`**, per the standing rule.
- `ARCHITECTURE.md`'s `items.ts` row and its invariant updated — it currently says skipping is
  "attribute-driven, never measured", which stays true and gets broader.
