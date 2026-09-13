# COPY-5 — `v-copy`: copy the user's current text selection

Owner, 2026-09-06: *"v-copy is okay ... I believe v-copy will be okay if it allows user selection
copied."*

Distinct from SEL-2. **SEL-2** copies a selection the *directive* made programmatically, inside
`v-select-text`. **This** copies whatever the *user* highlighted with their mouse or keyboard —
a copy button that respects what you just dragged over.

## The crux — the click destroys the selection

`mousedown` on a button collapses the document selection, or moves focus, **before** the click
handler runs. So a naive `v-copy="{ source: 'selection' }"` reads `getSelection()` and finds it
empty or collapsed. Browsers differ; this is only observable in a real one.

Solve this first, and say which approach you took and why:
- capture the selection on `mousedown`/`pointerdown` (before it is lost) and use the captured value
  when the click arrives — beware a mousedown that never becomes a click, and a selection that
  legitimately changes between the two;
- `preventDefault()` on `mousedown` so focus never moves and the selection survives — but that also
  suppresses focus on the trigger, which has a11y consequences worth stating;
- something better.

**Whatever you choose, the keyboard path must work too** — Enter/Space on a focused trigger, per
`v-copy/src/events.ts`, which already injects `tabindex`/`role`/key handling. A selection made with
Shift+Arrow and copied with Enter is a real flow.

## What gets copied

**`getSelection().toString()` is correct here** — and note this is the *opposite* of SEL-2's
conclusion, deliberately. There, the package had a competing resolved view and reporting one string
while copying another was the bug. Here there is no competing view: the user's selection **is** the
truth, and `toString()` is exactly what ⌘C would have produced. Say so in the code comment so a
future reader does not "fix" it into agreement with `v-select-text`.

Consequences to pin:
- A `user-select: none` region stringifies to `""`. **Refuse an empty selection** rather than wiping
  the clipboard — same rule as SEL-2's `'empty'`.
- Engines insert their own separators at block and cell boundaries. That is what the user would have
  got from ⌘C, so it is right. Document it; do not normalize it.

## Scope of the selection

Decide and justify: does it copy the **whole document selection**, or only a selection whose range
is inside the bound element (or a configured container)? Scoped is probably right — a copy button
inside a card should copy that card's selection, not something highlighted elsewhere on the page —
but a page-level "copy what I selected" button is also a real use. Recommend a default and say what
the escape hatch is.

## Surface

Fit it to the existing option vocabulary — `v-copy` already has `source` accepting a string, a
number and a getter (`src/resolve.ts`, `resolveText`). A `'selection'` sentinel, a `selection: true`
flag, or a getter recipe are all candidates; pick one and justify against the current shape. The
package's default source is trimmed `textContent`, so this is an alternative source, not a new mode.

**Composes for free with what already exists** — history, `dedupe`, `max`, `rich`, the feedback
window, the `aria-live` announce and the picker card all take the copied string and do not care
where it came from. Selections landing in the history picker is a genuinely good combination; check
it works and consider a card for it.

## Acceptance

- TDD. Baseline is **61 tests**; jsdom implements `Selection`/`Range` well enough for the string
  handling and the empty-refusal, but **not** for the mousedown-destroys-selection problem, which is
  the actual feature. Do not let a green jsdom suite stand in for that.
- **Browser-verified with trusted input**: `Input.dispatchMouseEvent` to drag-select real text, then
  click the trigger, then read the **real clipboard** back via `Browser.grantPermissions` +
  `readText()`. Do not use `cdp.mjs`'s `userGesture: true` path — it fakes the activation under test.
  Prove: a dragged selection is copied verbatim; the selection survives the click; an empty
  selection copies nothing and leaves a pre-seeded clipboard intact; keyboard selection + Enter
  works; and the copied string lands in the history with `dedupe` applied.
- **A negative control**: break the selection-capture and confirm the check goes red. Three agents
  this session found their tests weaker than they looked; one caught a check passing for the wrong
  reason.
- A playground card, per `tickets/_STANDARDS.md`. README section. Both must state the engine
  differences honestly rather than claiming uniform behaviour.

**Blocked by AUDIT-1** — `v-copy` is reported working by the owner, but no independent pass has
confirmed the other 12 cards still work after COPY-4. Do not build on an unaudited base.
