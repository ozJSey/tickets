# KB-4 — hover optionally registers as active input, and the arrows continue from it

Owner, 2026-09-13:
> "keyboard navigation should allow hover to register as active input (optionally) because in real
> world scenarios we wanna highlight hover and keyboard nav should continue on it."

The behaviour: arrow to item 3, move the mouse over item 7, press ArrowDown — you land on item 8,
not item 4. Every serious menu and combobox does this; none of this portfolio's do.

**Opt-in**, per the owner. Hover silently moving the active item is wrong for a toolbar and right
for a menu.

## Existing seams — this is a new reason, not a new mechanism

- `roving.ts:activate(group, index, reason)` already carries **why** activation happened.
- `directive.ts` already tracks `pointerdown` to tell a click from a Tab in `focusin`.
- `resolve.ts:73` already has an opt-in `activedescendant` mode.

## The three real design problems

**1. Does hover move DOM focus?** In roving-tabindex mode, "active" *is* focus — so hover would
steal focus from wherever the user actually is. That is wrong: hovering a list while typing in a
filter input must not blur the input. In `activedescendant` mode there is no conflict, because
active and focus are already separate.

Decide, and say so in the README: does hover-as-input require `activedescendant`, is it allowed in
roving mode with focus following (and the filter-input case documented as a hazard), or does it
move a *visual* active marker in roving mode without moving focus — which means a third state
between "focused" and "not", and a `data-*` hook for it.

**2. The combobox case is the one that matters.** Type in an input, arrow through a listbox below
it, hover a row. Focus must stay in the input the whole time. This is precisely what
`activedescendant` exists for and is the strongest argument for scoping the feature to that mode.

**3. THE TRAP, and it is the reason this is not a ten-line feature.** Arrowing through a long list
scrolls it. The item under a *stationary* cursor changes, which fires `mouseover`, which yanks the
active item back to wherever the mouse happens to be. The user presses Down and the highlight jumps
somewhere else. Every menu library hits this.

The fix that works: **react to `mousemove`, not `mouseover`** — a real cursor movement, not the
document moving beneath a still cursor. Belt and braces: suppress hover activation for a short
window after a key event, and/or ignore a `mousemove` whose coordinates have not actually changed.

**Do not ship this without a browser regression for it.** It is invisible to jsdom, invisible to
unit tests, and it is the single most likely thing to be wrong.

## Also decide

- **Does hover scroll?** It must not. `scroll.ts` runs on activation; a hover-driven activation that
  scrolls fights the mouse and can cascade. Wire the reason through so `'pointer'` skips the scroll.
- **Does hover fire the same event/state as a key?** Consumers styling `[data-…-active]` probably
  want one marker; consumers reacting to navigation may want to distinguish. `reason` already
  exists — surface it.
- **Leaving the group.** Does the active item stay where the mouse left it, or revert to the last
  keyboard position? Staying is what menus do.
- **Touch.** A `mousemove` is synthesised on tap by some browsers. Confirm this does not make a tap
  activate a neighbour.

## Acceptance

- Opt-in, defaulted off, named against the `focusgroup` vocabulary the package already adopted.
- **The scroll-under-stationary-cursor case has a browser regression that fails before the fix.**
- The combobox shape is a playground card: real typing in an input, arrows moving a listbox
  selection, hover taking over, focus never leaving the input — verified with real
  `Input.dispatchKeyEvent` and real `Input.dispatchMouseEvent`.
- Mutation-tested. Baseline is 162 tests and 50/50 mutants — and note that none of the previous 39
  mutants touched container resolution or skipping, which is how 0.1.0 shipped a live bug with a
  green run. Aim mutants at the hover path specifically.
- The one-tabbable invariant survives: hover must never leave the group with zero tab stops.
- README, CHANGELOG, and a line in the manual screen-reader walkthrough — hover is meaningless to a
  screen-reader user, so the keyboard path must remain complete on its own.
