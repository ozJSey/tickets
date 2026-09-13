# PG-15 — `smoke:dist` silently tests source for any scoped package

Found by the `v-keyboard-navigation` scaffold, 2026-09-06, while deciding its own alias key.

`distEntry()` in `playground/vite.config.ts` resolves a package's built entry by reading
`../<alias-key>/package.json`. The alias key for a scoped package is its **npm name** —
`@ozjsey/v-fit-children` — so the lookup becomes `../@ozjsey/v-fit-children/package.json`, which
does not exist. The failure is **silent**: it falls back to the source entry.

**Consequence: `PLAYGROUND_TARGET=dist` has been testing *source* for `@ozjsey/v-fit-children`.**
That is the one package in this portfolio that is actually published, so the dist gate has been a
no-op for the only library where a stale or broken artifact reaches real users. `pnpm smoke:dist`
and `pnpm interactions:dist` both report green while proving nothing about that package's build.

It will hit every package as they publish under the scope — which, per the owner's decision, is all
of them.

## Fix

Decouple the alias key from the directory name. The directory is the source of truth on disk
(`v-fit-children/`); the specifier is what demos import (`@ozjsey/v-fit-children`). `distEntry()`
must resolve via the directory, not the specifier.

**Whatever the fix, it must fail loudly.** A missing `package.json` or a missing `module`/`main`
entry has to throw with the package named, not fall back to source. The silence is the defect —
the same shape as PG-14, where a version mismatch degraded quietly instead of erroring.

## Acceptance

- `PLAYGROUND_TARGET=dist` genuinely loads `@ozjsey/v-fit-children`'s built artifact. Prove it by
  deleting or corrupting that `dist/` and confirming the run **fails** rather than passing.
- The same holds for any package aliased under a scope.
- A missing or malformed built entry throws, naming the package.
- Re-run `smoke:dist` and `interactions:dist` afterwards: any check that was passing against source
  in dist mode is a false green and must be re-judged.

## Related

- **PG-14** — the playground compiles demos with a different Vue than it runs. Same family: an
  infrastructure mismatch that degrades silently and invalidates verification done through it.
- `CLAUDE.md` records `dist/` going stale silently three times. This is why the dist gate exists,
  and it has not been running for the published package.
