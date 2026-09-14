# TT-22 — eight defects found by blind certification of `v-teleport-to` 1.1.0 (live on npm)

Found 2026-09-14 by an agent given the audit method and forbidden from reading `tickets/TT-*`,
`tickets/quality/` or `DESIGNS.md`. ~700 measured configurations read back off live rects, computed
styles and `data-*`, against **both** `pnpm dev` and `pnpm dev:dist`.

**Published at 1.1.0, and it is the only version.** Remedy is a `1.1.1` patch, not a hold.

**Verdict: partly works.** The *directive* is the strongest thing measured in this portfolio — see
"What could not be broken". The **composable is defective on its default path**.

---

## 1. P0 — `useTeleportTo` measures the pre-render DOM, and never self-corrects

`src/use-teleport-to.ts:274-282` runs the recalc in `watchEffect(…, { flush: 'sync' })`. Sync flush
fires the instant a dependency changes — **before Vue patches the DOM** — so the reference rect and
the host-measurement probe both read the previous frame's layout. There is no post-flush re-run, so
the wrong answer sticks.

The existing comment states the assumption that is wrong:

```ts
// Sync flush so consumers see updated styles immediately when they mutate the
// options ref — there's no separate render step to wait for here.
```

There is a separate render step to wait for, whenever the host's content is reactive.

Identical markup, identical options, 1280×660, 115px below the trigger and 513px above:

| | placement | fit | maxHeight | contentHeight | truncated | rendered host |
|---|---|---|---|---|---|---|
| **directive** | `top` | `flipped` | 240 | 242.5 | `true` | 240px, all 8 rows reachable |
| **composable** | `bottom` | `fits` | 115 | **10** | **false** | **114.6px sliver, 3 of 8 rows** |

The popover is cut to a sliver **and every signal it hands you says it is fine** — the exact failure
mode TT-17/18/19 were filed for, fixed for the directive, still open for the composable. It does not
recover after 1.8s or across close/reopen. One manual `update()` flips it to
`top / flipped / 242.5 / truncated: true`.

**Trigger condition:** the host's content or the reference's box changes in the same reactive tick
as an option. That is the ordinary dropdown shape — `enabled: open` plus `v-if` on the contents,
which is what the README's own *"Unmounting the host"* section advises. Also an async list, or a
`:class` toggle.

**Second face, same cause:** a reference that *moves* through reactive state is never picked up.
Measured — trigger moves 205px, the composable's host stays put indefinitely; the directive tracks
it, because its `updated` hook is post-flush.

- **Reproduces identically against the published `dist`.**
- `autoUpdate: true` incidentally masks both faces (the observer callback lands post-patch). The
  README presents `autoUpdate` as being for changes scroll/resize miss, never as required for the
  composable to see its own host — and it is **off by default**.
- **Why nothing caught it:** the 836 unit tests are jsdom, which has no layout; `11-composable.vue`
  is the one card with **no** interaction check; and that card's own shape — content always
  rendered, reference never moves — cannot express the trigger condition. `v-teleport-to` has the
  thinnest interaction coverage in the portfolio, 3 of 14 cards.
- **Why it matters most:** the composable is the README's prescribed workaround for transformed /
  `contain` ancestors. Anyone with a transformed app shell is told to take this path.

**Fix direction:** `flush: 'post'`, or keep sync for option-only changes and add a post-flush
re-measure. Whichever lands, it needs an interaction check on card 11 — the absence of one is why
this survived — and a card shape that can actually express the trigger (reactive content, moving
reference).

## 2. P1 — the README's primary Usage snippet does not run

`README.md:65` — `v-for="item in items"` with no `items` declared in `<script setup>`. Pasted
verbatim: `[Vue warn]: Property "items" was accessed during render but is not defined on instance`,
and the dropdown renders as an empty box. **This is the first code a new consumer copies.**

`pnpm docs:check` already flags it (`TEMPLATE_UNDECLARED:65`) but only as an advisory, so the gate
stays green. Worth asking whether `TEMPLATE_UNDECLARED` should be an error in the *first* runnable
sample of a README.

## 3. P1 — the README's two headline recipes cancel each other out

Usage uses `v-show="open"`. The Data-attributes section sells `data-teleport-state` as *"free
open/close animations without a separate `<Transition>` wrapper"* with the
`[data-teleport-state="open"] { opacity: 1 }` recipe.

Combined verbatim, the attribute is `"open"` from the first successful calc and **never leaves it** —
measured `opacity: 1` at mount-closed, at open, and at close. The transition never runs. `"closed"`
only appears for `enabled: false` or a missing `to`. Every demo card that shows the animation passes
`enabled: open`; the README's directive Usage never does.

Fix: put `enabled` in the Usage snippet, or correct the animation claim. Not both readings can stand.

## 4. P2 — shipped package metadata points at a repository that 404s

`repository`, `homepage` and `bugs` all resolve to `github.com/ozJSey/vue-teleport-to` → **HTTP
404**. On npmjs.com the Repository and Homepage links are dead and **there is no working issue URL**.

Consequently `README.md`'s `[ARCHITECTURE.md](./ARCHITECTURE.md)` resolves nowhere: `files: ["dist"]`
keeps it out of the tarball (`npm pack --dry-run`: 9 files, no `ARCHITECTURE.md`) and the repo it
would fall back to does not exist. `pnpm docs:check` reports `LINK_NOT_PACKED`.

**This is portfolio-wide — see `META-1`.** Seven of eleven published packages are affected, and it
needs an owner decision, so do not invent a URL here.

The 16 playground card deep links and the playground site itself are live (200).

## 5. P2 — the `maxWidth` × `overflow: 'shift'` paragraph documents behaviour the code does not have

README claims a numeric `maxWidth` makes the shift clamp use that value as the effective width, and
a CSS-string `maxWidth` makes it fall back to `parentWidth × widthMultiplier`.

Measured with `widthMultiplier: 20` and a 518px host: `maxWidth: 600`, `'600px'` and
`'min(90vw, 600px)'` **all** clamp to `left: 762` = `1280 − 518`, the real measured width in every
case. Both documented rules are wrong — **the implementation is better than its docs** — and the
advice to *"pass a number when overflow accuracy matters"* is obsolete. Fix the prose, not the code.

## 6. P3 — `data-teleport-truncated` / `data-teleport-collapsed` survive dormancy

DOM at `enabled: false`: `{ teleportState: "closed", teleportTruncated: "" }`. `src/directive.ts`
`updated` deletes `teleportPlacement` and `teleportFit` but not these two, so the README's own recipe
`.dropdown[data-teleport-collapsed] { display: none }` keeps matching a disabled host.

## 7. P3 — `overflow: 'shift'` is provably inert for horizontal placement

`calculate-position.ts:530-546`: for `placeRight` the host already starts at `minLeft = parentRight`,
so the clamp into `[parentRight, viewportWidth − hostWidth]` cannot move it; any overflow makes
`maxLeft < minLeft` and pins it where it already was. Symmetric for `'left'`. At `offsetX: 0` — i.e.
always, in practice — it is a no-op.

Measured on card 05: `placement: right` × `overflow: none | shift` produce byte-identical boxes at
every rail position and both multipliers (e.g. `1018…1525` of 1280 — 245px off screen — under both).

The README's caveat admits the no-overlap constraint wins, but the sentence before it promises a
clamp that can never happen, and **card 05's `overflow` control is dead in its `right` mode.**

## 8. P3 — three demo claims are not reproducible at the size the card is read at

- `13-content-measurement.vue`: *"Drag content … and the placement still moves."* At 1280×900 the
  placement never moves across the entire slider for `top`, `bottom` **or** `auto` (15/15 rows). It
  does move at 1280×560. **The card's central claim is invisible on a normal laptop window.**
- `09-virtual-reference.vue`: *"right-click near the bottom of the box and the menu opens above the
  cursor."* At 900px and 700px viewport heights it stays below (263px of room remains). Flips at 420px.
- `02-placement-flip.vue` (source comment): *"horizontal space actually varies, so the horizontal
  flip is reachable."* The only slider moves the reference **vertically**; horizontal space is
  constant, and `left` never flips in 30/30 rows — only `right → left` is reachable.

---

## What could not be broken — the TT-17/18/19 repair is sound

Pushed hard at it; nothing it claimed is broken.

- **Fit ladder**: 150 combinations on card 02 (5 placements × flip × 5 reference positions × 3
  content sizes). Every chosen side geometrically correct, never overlapping the trigger;
  `fits`/`flipped`/`neither`/`unmeasured` all reachable and consistent with reported spaces.
  `flip: false` pins the side.
- **Content measurement**: collapsing (`max-height: 0 !important`) and opacity closed states reach
  byte-identical verdicts across 30 combinations. `v-show` before vs after the directive agree.
  `truncated` appears exactly when `contentHeight > maxHeight`.
- **The livelock guard holds.** Built the shape the README warns about — content 120px taller on
  `top` than `bottom`, plus a collapsing animated closed state — swept the reference through the
  boundary and toggled open/closed 12 times in 840ms. 15 ticks total, **0 ticks in every idle
  window**, no oscillation, no recursive-update error.
- **Arrow**: 60 combinations across four sides, right-anchored and shifted hosts included — centre
  within 0.1px of the reference centre every time, and it tracks `overflow: 'shift'`.
- `matchWidth` > `maxWidth` > `widthMultiplier` > mobile full-bleed precedence; `maxWidth: 0`
  collapsing to the 21px `min-width` floor; `autoUpdate` / `autoUpdateSubtree` (drift 0 vs −61px
  across five scenarios); `scrollContainer` narrowed × `hideWhenReferenceHidden`; virtual reference;
  `strategy: 'absolute'`; `crossAxisAlign`; offsets; `zIndex`; `enabled` toggling.
- Horizontal `crossAxisAlign: 'center'`/`'end'` are **better** than documented — they use the real
  measured height, not the `maxHeight` proxy the README describes.
- Bare binding is clean across eight viewport positions: never off-screen, never overlapping, no
  console output.

**One instrument self-correction the certifier recorded:** its first probe chained `console.warn`
wrappers across iterations and appeared to show *"9 of 14 cards emit a spurious `to is required`
warning"*. That was the harness, not the library. Re-measured cleanly, only `09-virtual-reference`
warns, and legitimately — its `to` is genuinely null until you right-click.

## Suggested release — `1.1.1`

1. The composable's `flush: 'sync'` defect (finding 1) — or, failing a fix, a loud README warning
   that `useTeleportTo` needs `autoUpdate: true` or an explicit `update()` whenever the host's
   content or the reference's position is reactive.
2. `items` in the README Usage snippet.
3. Either `enabled` in the Usage snippet, or a correction to the `data-teleport-state` animation claim.
4. A working `repository` / `homepage` / `bugs` URL, and `ARCHITECTURE.md` either packed or unlinked
   — **blocked on `META-1`.**

**The directive alone would ship without hesitation.** The composable should not be recommended to
anyone until (1) lands — and since it is the documented escape hatch for transformed ancestors, that
is not a corner of the API.
