# DZ-7 — `maxCount` is documented as a total and implemented as a per-drop limit

P2. Quality-audit finding 6, deferred out of the 0.1.1 patch with reason. **The docs half is
already fixed** (README + `types.ts` now describe the per-event behaviour); this ticket is the
open *semantic* decision, which is a behaviour change and does not belong in a patch on a live
package.

## What it does today

`validate.ts:47` — `files.length > opts.maxCount` — compares only the current event's file list.
Nothing consults `instance.records` or anything the zone already holds. With `maxCount: 2`, two
successive two-file drops accept all four files and fire zero `onReject`. The same applies to
`multiple: false`, which permits an unlimited number of single-file drops.

Under `autoUpload: false` — the mode the README's recipe 8 and playground demo 08 build a review
queue with — `api.pending` therefore grows past `maxCount` without limit, and the consumer's
server-side cap is the only thing left enforcing it.

## The call to make

1. **Leave it per-event and keep the corrected docs.** Cheapest, already true, and defensible:
   validation is a property of an *event*, and the zone does not own the consumer's list.
2. **Make it a running total against `api.pending` + `api.uploading` + `api.failed`.** What the
   README said for four months and what "count cap" reads as. Breaking for anyone whose zone
   accepts repeat drops today, so it is a minor at least, and arguably a major.
3. **Both, named separately** — keep `maxCount` per-event, add `maxTotal` for the running cap.
   No breakage, and each name says what it does. This is the option to beat.

Whichever wins, `multiple: false` has to follow the same rule or the two options disagree.

## Acceptance

- Decision recorded here with its reasoning before any code moves.
- Browser check on demo 02 (validation) and demo 08 (queue) driving two successive drops.
- If the semantics change, `CHANGELOG.md` gets it under **Changed**, not **Fixed**, and the version
  is bumped accordingly.
