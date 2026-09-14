# META-1 — seven published packages point at a repository that does not exist

**Blocked on an owner decision.** Do not invent a URL.

Found while verifying a `v-teleport-to` certification finding (TT-22 §4), then measured across the
whole portfolio on 2026-09-14.

## What is wrong

`repository`, `homepage` and `bugs` in `package.json` are what npmjs.com renders as the
**Repository** and **Homepage** links on a package page, and they are the only route a user has to
report a bug. For seven of eleven published packages they resolve to nothing.

| Package | `repository` URL | |
|---|---|---|
| `v-copy` | `github.com/ozJSey/vue-copy` | **404** |
| `v-dropzone` | `github.com/ozJSey/vue-dropzone` | **404** |
| `v-keyboard-navigation` | `github.com/ozJSey/vue-keyboard-navigation` | **404** |
| `v-observe` | `github.com/ozJSey/vue-observe` | **404** |
| `v-scroll-into-view` | `github.com/ozJSey/vue-scroll-into-view` | **404** |
| `v-select-text` | `github.com/ozJSey/vue-select-text` | **404** |
| `v-teleport-to` | `github.com/ozJSey/vue-teleport-to` | **404** |
| `v-fit-children` | `github.com/ozJSey/vue-fit-children` | 200 |
| `vue-write-behind` | `github.com/ozJSey/vue-write-behind` | 200 |
| `bigdecimal-string` | `github.com/ozJSey/big_decimal_string` | 200 |
| `dependency-grouper` | `github.com/ozjsey/dependency-grouper` | 200 |

The pattern is exact: **the four that work are the four that have their own git remote** (they are
nested git repositories, see the root `.gitignore`). The seven that fail have never had a repo — the
URL was written to a convention rather than to something that exists.

## Why it is not cosmetic

- Every one of these is **live on npm right now**, so seven package pages carry two dead links.
- There is **no working issue URL**, so a user who hits a bug has nowhere to report it. For a
  portfolio whose stated distribution model is copy-paste, and whose own standard is *"mistake is a
  problem for not only us, but many many potential developers"*, that is the wrong end to be
  missing.
- It breaks in-README relative links as a side effect. `v-teleport-to/README.md` links
  `[ARCHITECTURE.md](./ARCHITECTURE.md)`; `files: ["dist"]` keeps it out of the tarball, so the link
  falls back to the repository — which does not exist. `pnpm docs:check` reports this as
  `LINK_NOT_PACKED` for several packages.

## Why this is blocked rather than dispatched

The root repository is `github.com/ozJSey/os_ideas`, and it **404s anonymously while accepting
pushes** — i.e. it exists and is **private**. So there is currently nowhere public to point these.

Every way forward is the owner's call, and two of them are outward-facing:

1. **Create seven public repos**, one per package, matching the existing convention. Most work,
   matches the four that already work, gives each package a real issue tracker.
2. **Make the portfolio repo public** and point all seven at it with a `directory` field —
   `{"type":"git","url":"…/os_ideas.git","directory":"v-teleport-to"}`, which npm understands and
   renders correctly. One decision, one repo. But it publishes `instructions/`, `tickets/`,
   `BOARD.md` and the whole run log along with the source.
3. **Point them at the playground repo** (`ozjsey/npm-portfolio-playground`, already public — it is
   what every README's live-demo link uses). Honest about where the demos are, dishonest as a
   "repository" for source that is not there.
4. **Remove the fields entirely.** npm then renders no Repository/Homepage link at all, which is
   accurate rather than broken, and is strictly better than a 404. A `bugs` email address is
   possible instead — `bugs` accepts `{"email": "…"}`.

Option 2 is the least work for the most coverage, but only the owner can weigh publishing the
planning material. Option 4 is the only one that needs no new infrastructure and is still an
improvement on today.

## When unblocked

- Fix all seven in one pass; do not leave a split convention.
- Re-run `pnpm docs:check` — several `LINK_NOT_PACKED` advisories resolve with it, and any that
  remain need `ARCHITECTURE.md` either added to `files` or unlinked from the README.
- Whatever is chosen, the four that currently work must end up consistent with the seven.
- Version bump: metadata-only, so a patch release per package.
