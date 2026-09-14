# DZ-4 — `--dropzone-*` CSS vars persist through the `rejected` state

P2. Audit finding, 2026-09-06. **Orphaned from DZ-3's rider list when that ticket closed** — filed
separately so it survives.

After a settled batch (`--dropzone-progress: 100`, `--dropzone-files-pending: 0`), a rejected drop
moves the state to `rejected` but leaves both vars **at 100/0 for the full `rejectDuration`**.
A rejection on a fresh zone is clean, so it is specifically the stale-value case.

The README says the vars "clear when the host returns to idle" and that "rejected drops do not touch
the vars". Neither is true: `rejected` is an undocumented fourth persist state.

Demo 9 only escapes visually because it has no `.dz[data-dropzone="rejected"] .bar` rule. **A
consumer who writes one sees a full progress bar on a drop that uploaded nothing** — a progress
indicator reporting completion for work that never happened.

## Acceptance

- Decide and document: do the vars clear on entering `rejected`, or is `rejected` a documented
  persist state? Clearing is the behaviour the README already promises.
- Browser regression, since this is a CSS-variable read on a real element.
- Do not regress the settled-batch or fresh-zone cases, both of which the audit confirmed correct.
- Baseline is 330 tests.

---

## RESOLVED — `@ozjsey/v-dropzone` 0.1.1, 2026-09-13

Folded into the 0.1.1 state-machine patch, since both are the same question ("what is this zone
showing, and does the reflection agree?") and the fix lives in the same three lines.

**Decision: the vars clear on entering `rejected`** — the behaviour the README already promised —
**unless files from an earlier batch are still in flight**, in which case they keep describing
those live requests. A rejection that lands during a real upload is not a stale-value case, and
blanking a moving progress bar for `rejectDuration` would have been a new defect of the same kind.

`setState` (`src/state.ts`) now drops the progress snapshot when the resolved state is `idle`, or
`rejected` with nothing unsettled in the snapshot.

A second fix fell out of the same read: the `rejected` auto-clear forced the zone to `idle`, which
reported an upload the rejection had interrupted as finished. It now hands back to
`nonDragRestState`, so the zone returns to `uploading` (or a sticky `error`) if that is what it
actually is.

- Unit: three tests in `vDropzone.test.ts` — the stale-value case, the live-upload case, and the
  auto-clear restoring `uploading`.
- Browser: `09-css-progress.vue` → "DZ-4: a rejected drop clears progress vars left by a settled
  batch", which also asserts the auto-clear returns to the sticky `error`.
- Neither the settled-batch nor the fresh-zone case regressed (both were already covered).
