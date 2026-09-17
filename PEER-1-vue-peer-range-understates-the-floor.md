# PEER-1 — three published packages declare `vue ^3.0.0` and cannot run on 3.0 or 3.1

**Found 2026-09-17**, chasing a vue-write-behind audit finding; the planner then swept all nine
v-* packages and found the same defect in four. This ticket covers the **three already-published**
ones. vue-write-behind's copy is fixed in WBV-1 (its own ticket, so two agents never edit one package).

## The defect

| package | peerDependencies | uses |
|---|---|---|
| `@ozjsey/v-scroll-into-view` 1.3.1 | `vue ^3.0.0` | Vue 3.2+ API |
| `@ozjsey/v-select-text` 1.0.0 | `vue ^3.0.0` | Vue 3.2+ API |
| `@ozjsey/v-teleport-to` 1.1.1 | `vue ^3.0.0` | Vue 3.2+ API |

`getCurrentScope`, `onScopeDispose` and `effectScope` all landed in **Vue 3.2.0**. A consumer on
3.0.x or 3.1.x installs cleanly — the declared range says they are supported — and then the import
throws `SyntaxError: Named export 'getCurrentScope' not found`. On a bundler resolving the
esm-bundler entry it is a **build-time** failure instead. Either way the package never worked on
the versions it advertises.

The audit that found this in vue-write-behind also measured the export tables:

```
vue 3.0.11   getCurrentScope=undefined onScopeDispose=undefined effectScope=undefined
vue 3.1.5    getCurrentScope=undefined onScopeDispose=undefined effectScope=undefined
vue 3.2.0    getCurrentScope=function  onScopeDispose=function  effectScope=function
```

## Work, per package

1. **Establish the real floor by reading the code, not by copying this ticket.** Find every Vue API
   the package imports and the version each was introduced in; the floor is the highest. Do not
   assume 3.2.0 — a package may use something later. Report the API that sets the floor.
2. Correct `peerDependencies.vue` to that floor.
3. Check for the same lie elsewhere in the package: a comment, a README compatibility line, or a
   test-matrix config asserting the old range. In vue-write-behind the matrix claimed to *prove*
   the range while its lowest version sat above the first version that had the API — a gate written
   so it could not fail. Look for that shape here.
4. CHANGELOG entry stating plainly what the range was, what it is, and that the old range was never
   satisfiable. Do not describe it as dropping support for versions that never worked.

## Version

**Patch for each** (owner rule: the MAJOR never moves). Narrowing a peer range looks like a
breaking change, but these versions were never functional — this corrects a false claim rather
than removing a platform. Say exactly that in the CHANGELOG.

The publish order is already handled: all three are in `scripts/publish.mjs`'s hard-coded `ORDER`,
so a version bump is all it takes for the script to pick them up in the right position.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **Evidence the floor is right, per package**: name the API, the file:line it is imported at, and
   the Vue version that introduced it.
2. **A real install check on the declared floor.** Install the package's built tarball against the
   lowest version the corrected range allows and import it — it must work. Then do the same one
   minor *below* the floor and show it failing. That pair is the negative control; without the
   failing half this ticket cannot be believed.
3. Each package's existing suite stays green — report real numbers per package.
4. If a package's test matrix claims to cover the peer range, either make it actually run the floor
   version or delete the claim. Do not leave a gate that cannot fail.
5. `node scripts/changelog-audit.mjs` stays clean.
6. Do not run `npm publish`. Do not commit. Leave the work in the tree.
