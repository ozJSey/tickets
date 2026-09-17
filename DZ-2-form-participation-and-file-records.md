# DZ-2 — v-dropzone: native form participation + reactive per-file records

**Owner, 2026-09-16 (audit walkthrough, stop 7):** build A and B below; `dropOn: 'document'`
parked, not killed. One agent for both — they meet in the records store and must stay coherent
(removing a record has to update the form's FileList too).

## A. `name` — dropped and pasted files ride a plain `<form>` submit

Every dropzone library breaks the oldest contract on the web: files dropped inside a
`<form method=post enctype=multipart/form-data>` silently vanish from the submit
(react-dropzone #880 tells users to hand-roll a hidden-input sync; FilePond's plugin base64s into
hidden inputs and its own docs admit mobile OOM). v-dropzone already owns a real
`<input type=file>` inside every zone — so:

- `name: 'attachments'` puts the name on that input and syncs **validation-accepted** files from
  drop, paste, and pick into its `FileList` via the `DataTransfer` constructor
  (`input.files = dt.files`).
- Native behaviours come free and must be pinned, not assumed: form `reset` clears the zone's
  records; `required` participates in constraint validation; `multiple` follows the existing
  maxFiles semantics.
- Rejected files never enter the FileList — the sync source is the accepted-records projection,
  not the raw drop.

## B. `dz.files` — the file list every consumer hand-builds

Project the instance's record store (ARCHITECTURE: "one store, everything else is a projection")
into one consumer-facing reactive array:

```ts
{ id, file, status: 'pending'|'uploading'|'done'|'error', percent, previewUrl?, response?, error? }
```

- `previewUrl` only for `image/*`, created lazily, **revoked by the directive** on record prune,
  replacement, and unbind — the `createObjectURL` leak class dies here, and a test must prove
  revocation actually happens (spy in units; browser check that the `<img>` renders).
- `dz.remove(record)` — the missing lever for settled records. **Interlock with A:** removing a
  record re-syncs the FileList in the same tick; a form submitted immediately after a remove
  must not carry the removed file.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. Real-browser check for A that proves *participation*, not plumbing: drive a real drop via CDP
   input, then read `new FormData(form).getAll('attachments')` out of the live page — file names
   and sizes must match the accepted set. No actual navigation (FormData read-back, not submit).
   **Negative control:** same card without `name` → FormData comes back empty.
2. Reset/required/multiple pinned in the same card; reset's check reads the DOM state attribute
   AND the FileList length.
3. B's remove-interlock check: drop two files, `dz.remove` one, FormData carries exactly one.
4. Default-configuration rule applies (BOARD standing criteria): the existing bare-binding cards
   must stay green — `name` and `dz.files` change nothing when unused.
5. README: both features open on their cards (DOCS-4 convention); A leads with the no-JS form
   story — that is the wedge. Version: **patch** (owner rule 2026-09-16).
