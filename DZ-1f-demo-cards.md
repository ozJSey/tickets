# DZ-1f — `v-dropzone` demo cards after the default flip

**Urgent: the playground currently teaches the wrong thing.** DZ-1a landed `clickToPick`
default-on. **11 of the 12 cards silently gained click-to-pick** — only `03-click-to-pick.vue`
passes the option explicitly. Prose across the tab still describes drop-only behaviour.

Design: `DESIGNS.md` → DZ-1. Standards: `tickets/_STANDARDS.md`.

## Per card

- **`12-folder-drop.vue` — worst offender, fix first.** The card is entirely about folders and its
  prose says the click-to-pick input "would need `webkitdirectory`" — and clicking it now opens a
  **file** picker. Set `clickToPick: false` here. It doubles as the demo set's live opt-out example.
- **`03-click-to-pick.vue` — repurpose.** `clickToPick: true` is now redundant. Make it the
  guard-and-keyboard card: it already has a `<button>`, an `<a>` and a checkbox inside the zone.
  Add a paragraph of selectable prose with "select this text and let go — no dialog", a
  `[role="button"]` chip exempted via `clickIgnore`, a `clickToPick` toggle, and a
  `:focus-within` ring with "press Tab, then Enter". One card carries the whole change.
  Update `manifest.ts` title/blurb/tags.
- **`09-css-progress.vue`** — `.bar` and `.counter` live inside the zone; clicking a *progress
  readout* mid-upload opens an OS dialog. Add `clickIgnore: '.bar, .counter'`. Best argument in
  the set for the option existing.
- **`10-state-machine.vue`** — the zone renders `dz?.state`; clicking to read it opens a picker
  and injects entries into the state trail. Guard it.
- **`11-enabled.vue`** — three `v-for` zones each containing a `<ul><li>` of filenames; clicking a
  filename opens a dialog. Guard it. Also the best proof that `enabled: false` destroys the input
  and its tab stop — say so.
- **`04-paste.vue`** — drop the manual `tabindex="0"` on `.dz` and switch its rule to
  `:focus-within`; the directive's focusable input now satisfies the `pasteOn: 'host'` focus
  requirement by itself. Zone also contains `<img>` previews → stray-click surface.
- **`07-api.vue`** — closing prose at `:71` teaches "`open()` works with no `clickToPick`". Reword.
- **`01`, `02`, `05`, `06`, `08`** — add a click sentence and `cursor: pointer`. `02` gains a real
  bonus worth stating: clicking now opens a picker honouring the live `accept` input.

Every card needs `cursor: pointer` and a visible focus ring — the picker input is 1px and clipped,
so the **host** must render focus. `.dz:has(> input[type="file"]:focus-visible)` for precision,
`.dz:focus-within` as the broad fallback. Both pure CSS; the directive must not paint.

## Acceptance

- No card's prose contradicts its behaviour. Grep the tab for "drop" claims that are now
  "drop or click".
- From `playground/`: `pnpm typecheck`, `pnpm smoke`, `pnpm smoke:dist` green — **and look at a
  screenshot**, per `CONVENTIONS.md`; smoke passing is not correctness.
- Drive the changed cards over CDP (`playground/scripts/lib/cdp.mjs`) and prove the guards fire on
  the real cards, not just in isolation: a click on `09`'s progress bar opens nothing; a click on
  `11`'s filename opens nothing; `12` opens nothing at all.
- **Do not edit `playground/scripts/interactions/v-dropzone.mjs`** — under migration. It currently
  runs 73/74; the one failure is the known `display: none` assertion at `:257`. Report if your
  changes alter that count.

## Carry-over from DZ-1a/d

A 13-check CDP harness was written to the scratchpad at
`.../scratchpad/dzverify/{page.html,verify.mjs}` (accepts `DZ_DIST=<path>`), including the
negative control. **The scratchpad is disposable** — if the harness migration has landed by the
time you run, port it; otherwise note it in your report so DZ-1g can.
