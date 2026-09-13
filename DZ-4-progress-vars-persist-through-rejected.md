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
