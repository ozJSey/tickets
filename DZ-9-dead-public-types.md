# DZ-9 — remove `UploadProgressEvent` and `UploadResult` at the next major

P2. Quality-audit finding 8. Both are marked `@deprecated` as of 0.1.1 and the README no longer
advertises them as usable; deleting them is breaking, so it waits.

Both are re-exported from `src/index.ts` and were listed in the README's "every consumer-facing
type is exported by name" block, but nothing in the package constructs or accepts either:
`onProgress` is `(file: File, percent: number) => void`, and no callback or return value is an
`UploadResult`.

`UploadProgressEvent` is worse than merely dead — its `{ file, loaded, total, percent }` shape is
exactly what a reader assumes `onProgress` receives, so the first thing a consumer writes is
`onProgress: (e: UploadProgressEvent) => …`, which does not compile.

## The call

Delete both at the next major, **or** make them real:

- `UploadProgressEvent` becomes the argument shape of a new `onProgressEvent`, with real
  `loaded`/`total`. Note the function transport cannot supply byte counts — it reports a percent —
  so the type would be URL-path-only, which is an argument for deleting it instead.
- `UploadResult` becomes the payload of a batch-settled callback (`onSettled(results)`), which is
  a genuinely missing piece of the API: today a consumer has to reconstruct "the batch is done"
  from `api.state`.

## Acceptance

- Removal (or the new API) lands in the same release as a major version bump and a `CHANGELOG.md`
  **Removed** / **Added** entry.
- `npm run typecheck` and `npm pack --dry-run` both clean; the `.d.ts` no longer names them.
