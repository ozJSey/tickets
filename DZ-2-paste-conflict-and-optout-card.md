# DZ-2 — `pasteOn: 'host'` vs the click-to-pick default, + a real opt-out card

Both raised by the owner, 2026-09-06:
> "Why did you remove copy paste example of v-dropzone and why did you not made an example of
> clicking disabled but you can still drop."

## Audit update, 2026-09-06 — partly answered, and worse than framed

The independent audit measured it: `pasteOn` defaults to `'host'`, a `<div>` host is not focusable,
and the only reliable focus target inside it is the directive's own picker input. So **`paste: true`
alone is close to dead** — with `pasteOn` omitted, a real ⌘V with a PNG on the clipboard does
nothing until the user Tabs in. Clicking the zone opens a file chooser and leaves
`document.activeElement === BODY`.

One nuance the audit found: after a real click, Chrome *does* route a later paste to the host via
the click's collapsed selection, so it is not 100% dead — but **Tab is the only deterministic
route**, and DZ-3 shows Tab currently scrolls the page away from the zone. The two bugs compound.

Demo 4 already handles this correctly with an explicit hint, so this is a **default and docs
problem, not a demo one** — which inverts Part A's original framing. The Options table flags nothing
on the `'host'` default row.

## Part A — a library-level conflict, not a demo bug. **Investigate before fixing.**

`pasteOn: 'host'` requires focus inside the host. Card `04-paste.vue` originally told the reader
*"you must click this box first"* — the natural gesture for focusing a paste target.

**Click-to-pick is now on by default, so clicking that box opens a file dialog instead.** The DZ-1f
agent responded by rewriting the instruction to "press Tab to focus this box first" and exempting
the previews with `clickIgnore: '.shots'`. That treats the symptom. The two features now fight,
and **no test caught it because each works perfectly alone.**

Answer these before changing anything:

1. **Where does focus actually land when a user clicks a `pasteOn: 'host'` zone?** The host click
   handler calls `input.click()` on the hidden file input. Does focus end up on that input, on the
   host, or nowhere? If it lands on the input, does a subsequent ⌘V still reach the host's paste
   listener, or is it swallowed by the file input? **Measure it in a real browser** — this is
   exactly the class of thing jsdom cannot answer.
2. **Is `pasteOn: 'host'` substantially broken by the default flip?** If clicking can no longer
   focus the zone and Tab is the only route, that is a real regression in a documented feature.
3. What is the right resolution? Options to weigh, not a menu to pick blindly:
   - the host click handler focuses the host (or the input) *in addition to* opening the picker,
     so click-to-focus keeps working;
   - `pasteOn: 'host'` implies a focus affordance of its own;
   - the two are documented as mutually awkward and one must be opted out of — the weakest answer,
     and only acceptable if the first two are worse;
   - something better.
4. Whatever the resolution, `04-paste.vue` must go back to a card a reader can use with the
   *obvious* gesture. Tab-to-focus as the primary instruction is a downgrade.

**Do not accept "it works if you press Tab" as done.**

## Part B — the missing card: click off, drop still on

`clickToPick: false` is currently demonstrated only as scenery on `12-folder-drop.vue` (a card about
folders) and behind a toggle on `03-click-to-pick.vue`. **No card has the opt-out as its subject.**
That was a planning error in the DZ-1f brief.

Add a card whose whole point is: **clicking does nothing, dropping still works.** It should make the
absence legible rather than invisible — no `cursor: pointer`, no focus ring, no picker input in the
DOM, no tab stop — and say why you would want that (a zone that is also a layout container, a
read-only region that still accepts drops, a card with its own Browse button doing `api.open()`).

Pair it with the fact DZ-1c established: with `clickToPick: false` the input survives only for
`api.open()`, and is then `tabindex="-1"` + `aria-hidden`, so it is not a phantom tab stop.

Decide placement: a new card, or promote it into `03-click-to-pick.vue` as an explicit second zone
side by side with the default one. **A side-by-side contrast is probably stronger than a toggle** —
a toggle shows one state at a time and the reader has to remember the other.

## Acceptance

- Part A's investigation is reported as findings **before** any fix is applied; if it is a library
  bug, it is fixed in `v-dropzone/src/` with tests, not worked around in the card.
- `04-paste.vue` works with the gesture a user would actually try.
- A card exists whose subject is the opt-out, and a reader can see what "off" looks like.
- Browser-verified per the standing rule in `BOARD.md`: drive the paste card over CDP, click the
  zone, and read back where `document.activeElement` is and whether a synthetic paste reaches the
  host. Use `Page.fileChooserOpened` for picker signals.
- `pnpm typecheck`, `pnpm smoke`, `pnpm smoke:dist` green, and look at a screenshot.

**Note:** DZ-1f's card edits are committed but were **never browser-verified** — that agent died at
its verification step twice. Treat the current state of all 12 cards as unproven, not as a baseline.
