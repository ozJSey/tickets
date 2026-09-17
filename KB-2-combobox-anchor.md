# KB-2 — combobox anchor: `aria-activedescendant` on the input, arrows forwarded (+ two P1s)

**Owner, 2026-09-16 (audit walkthrough, stop 9):** build the combobox anchor. Selection intents
parked, not killed. The two audit P1s ride along because one of them lives in the same file.

## A. The feature

The README documents this as a **known limitation** and ships a ~20-line hand-roll recipe: a
combobox needs `aria-activedescendant` on the input that holds focus, but the directive writes it
on the host, so today the consumer mirrors the attribute through an event handler AND forwards
↑/↓ through the imperative api themselves.

```ts
const options = { activedescendant: { input: '#city' }, hover: true }
```

```html
<input id="city" role="combobox" aria-controls="cities" />
<ul id="cities" role="listbox" v-keyboard-navigation="options">…</ul>
```

- The directive writes the active-descendant pointer onto the **named input**, not the host.
- It claims exactly **ArrowUp / ArrowDown / Home / End** on that input. **Never a printable key** —
  typing stays the consumer's, which is non-negotiable (`keys.ts` already states this stance; do
  not weaken it for convenience).
- Accepts a selector or an element/ref, resolved the same way `container` options are elsewhere in
  the portfolio. A selector that resolves to nothing is a dev warning, not a silent no-op.
- Teardown clears the attribute from the input — it is not the directive's element, so the cleanup
  path must reach outside the host. This is exactly the class of bug TT-22 §6 found in
  v-teleport-to (a dormant branch hand-listing attributes); use one shared clear helper.

## B. P1 — `generatedIds` pins every ever-active element until unmount

`src/roving.ts:416`. The audit calls it a slow leak **on the combobox path specifically**, which is
the path this ticket makes first-class — so it is fixed here, not deferred. Verify the lifetime
claim first, then bound it (drop ids for elements no longer in the item set), and pin with a test
that fails against the current code.

## C. P1 — published 0.3.0 has dead `repository`/`homepage`/`bugs` URLs

`package.json:9`. Local was corrected in `19d2c7d`; the **published artifact still 404s** and only
clears on the next publish. No URL is to be invented — this is META-1, the owner's call. Confirm
the local value resolves anonymously, and say in the completion note that the published page stays
wrong until 0.3.1 ships.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **Real-browser check**, not jsdom, for the whole loop: focus the input, drive real ↑/↓ key
   events through CDP, read `aria-activedescendant` off the **input** and the active item's state
   out of the live DOM. Type a printable character and assert the directive did **not** intercept it.
2. **Negative control:** remove the option, same card — the attribute never appears on the input.
3. Teardown check: unbind, assert the input is clean.
4. B gets its own failing-first test; C gets a resolution check, not a fix-by-inventing.
5. Default configuration stays green (standing rule) — bare `v-keyboard-navigation` is unaffected.
6. Release: **patch** (owner rule — major never moves).
