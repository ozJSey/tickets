# DZ-10 — playground demo 13 ships a `JSON.stringify`-diffed MutationObserver as the example

P2. Quality-audit finding 13. Not folded into 0.1.1 because the demo is *correct*; what is wrong
is what it teaches, and rewriting a card is a demo ticket, not a patch to a live library.

`playground/src/demos/v-dropzone/13-click-opt-out.vue:63` — `refresh()` serialises two probe
objects and string-compares them to decide whether to assign state, with a 12-line docblock
explaining that the guard exists because a MutationObserver on the zone used to hang the tab. That
is an honest record of a real incident (a microtask loop: mutation → reactive state → render →
`updated` → mutation).

The problem is the card's job. It is the flagship `clickToPick: false` example on a page whose
stated purpose is to show consumers how to use the directive, and half the file — the `Probe`
type, `probe()`, the observer, the `openViaApi` re-probe — is instrumentation for reading the
directive's own writes back out of the DOM, which `README:277` explicitly tells consumers *not* to
do. A stranger pastes the observer pattern, not the directive usage.

**The root cause is now gone**: 0.1.1 made `setState` idempotent, so `data-dropzone` no longer
queues a `MutationRecord` for an identical value, and `syncPickerAttrs` / `applyPickerA11y` were
already idempotent. The deep-compare guard is defending against a hazard the library no longer
has.

## Shape of the fix

- Drive the card's readout from `api.state` / `api.pending` through the `ref` option — the
  supported way to observe a zone — and delete the observer, the `Probe` type and the
  `JSON.stringify` compare.
- If the card genuinely needs to prove "there is no input element in the opted-out zone", that is
  a one-line `querySelector` on a button press, not a subtree observer.
- Keep the incident itself recorded — in `ARCHITECTURE.md`, where it already is, not in the
  copy-paste example.

## Acceptance

- The card still demonstrates what `clickToPick: false` does, bare binding first.
- The existing `13-click-opt-out.vue` interaction checks stay green, including the
  `Page.fileChooserOpened` one.
- Removing the guard does **not** hang the tab — verified by loading the card with the observer
  deleted, which is also the negative control for the idempotence fix.
