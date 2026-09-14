# DOCS-4 — every README deep-links to the specific card, not just the tab

Owner, 2026-09-14:
> "make sure all sub libraries have
> `https://ozjsey.github.io/npm-portfolio-playground/#v-teleport-to#their-subsection`
> linked clearly relatively soon on readme's"

Today every README links to its **tab**. He wants links to the **specific card** that demonstrates
the feature being documented, placed clearly and early.

## The URL form

`App.vue:84` already does `location.hash.replace(/^#/,'').split('/')` into `[id, sub]`, and `sub`
is currently only ever `'docs'`. **A card id fits the existing slot** — no new scheme needed:

```
#v-teleport-to                          the tab            (works today)
#v-teleport-to/docs                     the documentation  (works today)
#v-teleport-to/02-placement-flip.vue    a specific card    (this ticket)
```

Prefer `/` for consistency with `/docs`. **But also accept `#` as a separator**, because that is
the form the owner wrote and may share: `#v-teleport-to#02-placement-flip.vue` must resolve
identically. Decide whether the card segment is the filename or a cleaned slug — a slug reads far
better in a README and in an address bar, so prefer it, but then the mapping must be unambiguous
and stable, because **these links ship inside published packages and cannot be changed cheaply
once someone has the tarball.**

## Behaviour

Landing on a card link should select the tab **and** bring that card into view and make it obvious
which one was meant — scroll to it and mark it. Do not collapse or hide the others; a reader
following a link from a README usually wants the surrounding context too.

An unknown card id must fall back to the tab rather than a blank page, and say so — a stale link in
a published README is inevitable, and it should degrade to something useful.

## The READMEs

Each package's README gets deep links **early and clearly** — the owner's "relatively soon"
— pointing at the card for the feature under discussion, not one link to the tab at the top and
nothing after.

Judgement, not mechanics: a README section that documents `flip` should link to the flip card, the
`dedupe` section to the history picker, the upload recipes to the upload cards. **Do not paste a
link into every heading** — a wall of links reads as noise and gets skipped. Pick the sections
where seeing it run genuinely beats reading about it.

## Scope and coordination

All twelve packages. **Two have agents in them right now — `v-fit-children` and `v-copy`.** Leave
those two READMEs alone and report exactly what you would have added, so it can be applied after.

`dependency-grouper` and `vue-provide-seeker` have no tab, so they keep their index link.

**`vue-provide-seeker`'s link is currently broken and is live on npm**: it is written
`**[label]**(url)`, which is not link syntax — the URL renders as literal text on the package page.
Fix that while you are there.

## Acceptance

- Card deep-links resolve, with both `/` and `#` separators.
- An unknown card id degrades to the tab, visibly.
- `pnpm docs:check` passes — its link checker now validates these, and a link to a non-existent tab is
  already a finding it catches.
- `pnpm smoke` and `smoke:dist` green.
- A browser check that follows a deep link and asserts the right card is selected and in view.
  Negative-control it: point the check at a card that does not exist and confirm it fails.

---

## Addendum, 2026-09-14 — what landed, and the two READMEs held back

**The card segment is a slug, not the filename.** `02-placement-flip.vue` → `placement-flip`: the
ordering prefix and the `.vue` are stripped, and nothing else is. The rule lives in exactly one
place, `playground/src/card-link.ts`, with the reasoning written there. Short version: these links
ship inside tarballs and cannot be corrected for a reader who already has one, so the segment has to
be the part of the name least likely to change — and the two parts that churn for reasons unrelated
to the card's identity are the number (every insertion renumbers what follows) and the extension.
Uniqueness is enforced at registry build time as a `manifestProblems` entry, which renders as an
error banner and fails `pnpm smoke`, so a collision cannot ship quietly. Resolution is lenient
inbound — the raw filename and `#` as a separator both resolve — and canonical outbound.

**`vue-provide-seeker`'s link was already correct on disk** when this run reached it:
`> **[The rest of the portfolio](https://ozjsey.github.io/npm-portfolio-playground/)**`, which is
bold wrapping a link, not `**[…]**(…)`. `pnpm docs:check` reports no `MALFORMED_LINK` for it, and its
detector is negative-controlled every run. Nothing to change. `dependency-grouper` keeps the index
link as specified.

### Held back: `v-copy` and `v-fit-children`

Agents were live in both packages, so their READMEs were not touched. Everything else in this ticket
is done and gated; `pnpm docs:check` reports `NO_CARD_LINK` for exactly these two, which is the standing
record until the patches below are applied. Every slug named here was followed in a browser by
`pnpm deeplinks` and resolves.

#### `v-copy/README.md`

Replace line 3 with:

```md
See in action: [npm portfolio playground](https://ozjsey.github.io/npm-portfolio-playground/#v-copy).

**Or open the card for the thing you came for** — seventeen of them, all editable in the browser:
[bare binding](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/bare) ·
[copy history](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/history) ·
[the history picker](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/history-picker) ·
[aggregated multi-select copy](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/multi-select) ·
[copy the user's selection](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/user-selection) ·
[slot-like controller](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/controller)
```

Then, one blockquote directly under each of these headings:

| Heading | Line to insert |
|---|---|
| `## The part nothing else does: a clipboard history` | `> [Clipboard history picker — copy, re-open, copy again](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/history-picker) is this paragraph, running: copy a few values, open the teleported dropdown, pick one, and it goes back on the clipboard.` |
| `#### De-duplication (`dedupe`, on by default)` | `> [dedupe scope](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/dedupe-scope) — one row per payload or one row per label, side by side, with [the history picker](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/history-picker) for what promotion looks like in a real list.` |
| `### Copy what the user selected` | `> [Copy what the USER selected](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/user-selection) — highlight across elements and press the button; the selection survives the press.` |
| `#### Whose selection? `within`` | `> [Whose selection is it — scoping with `within`](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/selection-scope).` |
| `### Nothing to copy` | `> [Nothing to copy](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/nothing-to-copy) — an empty binding and a not-yet-loaded one, both refused rather than flashing "Copied!".` |
| `### Aggregated copy — many selected rows, one payload` | `> [Multi-select rows → one copied context](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/multi-select). This is the package's wedge; it is worth watching rather than reading.` |
| `## Accessibility` | `> [Accessibility](https://ozjsey.github.io/npm-portfolio-playground/#v-copy/a11y) — Tab to a non-interactive host and press Enter.` |

#### `v-fit-children/README.md`

Replace line 3 with:

```md
See in action: [npm portfolio playground](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children).

**Or go straight to the card** — drag the width slider on any of them:
[chips with a +N badge](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/basic) ·
[data mapping](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/data-mapping) ·
[pinned children](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/keep-visible) ·
[inline badge](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/inline-badge) ·
[the state attribute](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/state-attribute) ·
[the event contract](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/event-contract)
```

Then, one blockquote directly under each of these headings:

| Heading | Line to insert |
|---|---|
| `## Quick start` | `> [Chips with a +N more badge](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/basic) is this snippet with a width slider on it.` |
| `## Event` | `> [The event reports what is true now](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/event-contract) — starting at a width where nothing fits, so the first pass has to dispatch too — and [isOverflowing on a row that cannot hide anything](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/pinned-overflow).` |
| `## Keeping elements visible` | `> [Pinned children](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/keep-visible) — `keepVisibleEl` and `data-v-fit-keep` surviving the cull, and [v-show children are left alone](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/v-show) for the child you hid yourself.` |
| `## Data mapping` | `> [data → hiddenData + hiddenIndices](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/data-mapping), and [decorative separators outside the mapping](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/decorative) for the children that must not consume a data index.` |
| `## Inline "+N" badge` | `> [Inline badge with offsetNeededInPx: 0](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/inline-badge).` |
| `## Attributes` | `> [CSS-only styling via data-v-fit-state](https://ozjsey.github.io/npm-portfolio-playground/#v-fit-children/state-attribute) — style the overflow state without an event handler.` |

Applying either one clears that package's `NO_CARD_LINK` advisory. Re-run `cd playground && pnpm
deeplinks` afterwards: pass 3 harvests card links straight out of every sibling README, so a typo in
a slug fails the run by file and line.
