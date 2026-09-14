# DZ-12 — nothing bounds `records` / `progressBatch` on a long-lived page

P3. Quality-audit finding 5. **Materially reduced by 0.1.1** — a new drop now prunes failed
records, which was the main way the map grew without limit — but nothing *bounds* either
collection, so this stays open.

## What remains

- A failed record is removed by its own successful retry, `cancel`, `dismissError()`, the next
  drop/paste/pick that brings in accepted files, or unmount. A consumer who never wires a Dismiss
  button (the README presents it as optional) and never drops again still holds every `File` that
  failed — and its backing blob — for as long as the host element lives.
- `progressBatch` is a second hard reference to the same `FileRecord`s. It is cleared on a
  transition to `idle` and now also on a spent `rejected`, but the sticky `error` state by
  definition reaches neither until something clears the error.
- Nothing caps the size of either collection. On an upload widget living in an SPA shell across a
  long session on a flaky connection, memory climbs by the size of every file that ever failed.

`progressBatch` deliberately holds records already deleted from `records` — that is what lets the
CSS vars read 100% through the `success` window, and it is documented. The leak is that neither
collection has an upper bound.

## Options

- Drop the `File` reference from a failed record and keep only what `api.failed` and `retry()`
  need. Does not work: `retry()` needs the `File` to re-send it.
- Cap `records` at N failed entries, evicting oldest. Silent data loss from a public array.
- **Most likely**: leave the retention rule as it is and document the ceiling honestly — the zone
  holds every un-dismissed failure — plus make `dismissError()` prominent in the README's error
  recipe rather than optional. A memory profile over a 50-failure session would say whether
  anything stronger is warranted.

## Acceptance

- A measurement first: heap retained after N failed uploads, with and without a Dismiss button.
  This ticket should not produce code before it produces a number.
