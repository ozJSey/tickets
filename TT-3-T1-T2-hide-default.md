# TT-3 / T1+T2 — hide the host when the reference leaves the viewport, **by default**

This is the owner's original request: *"teleport-to lacks a severe functionality... if it's fully
hidden, we keep them up. Also window scrolling should be a default option to hide. They can opt
out if they choose to."*

Full design: `DESIGNS.md` → "TT-3". Standards: `tickets/_STANDARDS.md`.

**Unblocked.** T0 landed (the `display`/`v-show` stomp — hiding now goes through `visibility` +
a `data-teleport-hidden` ownership record) and TT-7 landed (fit-based flip; the placement decision
moved to a new `src/resolve-placement.ts`). **Baseline is now 698 tests and version 2.0.0** —
pull the current source before planning; `calculate-position.ts` has changed twice today.

## T1 — extend the predicate to the viewport, and make it 2D (default still off)

`src/calculate-position.ts:532` gates on `opts.hideWhenReferenceClipped === true && boundaryEl`.
`boundaryEl` is non-null only for an explicit connected `HTMLElement`, so **the default
`boundary: 'viewport'` can never trigger hiding** — that is the whole bug. `boundaryRect` already
falls back to the viewport rect, so the math works the moment the `&& boundaryEl` gate is dropped.

The predicate is also **axis-aware**, which is wrong for a visibility test — a reference that
scrolls fully out *horizontally* under `placement: 'bottom'` is never considered hidden. Replace:

```
clipRect = intersect(boundaryRect, viewportRect)
referenceHidden = refBottom <= clipRect.top  || refTop  >= clipRect.bottom
               || refRight  <= clipRect.left || refLeft >= clipRect.right
```

Binary "fully outside", not a ratio — the owner said "fully", and a ratio hides a dropdown whose
trigger is still usefully on screen. Four-sided. Intersected with the viewport so a reference
inside a `boundary` pane that has itself scrolled off screen counts as hidden. Both rects are
already computed; this adds **zero** DOM reads. Keep `clipRect` separate — do **not** feed it into
`boundaryRect`, which the placement/space math uses.

`<=` / `>=` (touching counts as hidden) matches the current code and Floating UI, and gives a
freebie: a `display: none` reference returns an all-zero rect and now hides, where today the host
silently teleports to (0,0). **Note the existing comment at `:530-531` claims the opposite and is
wrong** — fix the comment, keep the behaviour. The same wrong claim is in `v-teleport-to/TASKS.md:95`.

**Tests:** no `boundary` + reference above the viewport → hidden; `placement:'bottom'` + reference
fully left of the viewport → hidden; reference inside a pane that has scrolled off → hidden;
mobile width + reference below viewport → hidden; and **invert the three tests at `:4189`, `:4200`,
`:4209`** which currently assert the no-op that is being removed. Those three are the only fixtures
in the file placing a reference fully outside the viewport, so nothing else picks up an incidental hide.

## T2 — flip the default and rename

`hideWhenReferenceHidden: boolean`, **default `true`**. Named for the outcome, not the mechanism —
"clipped" reads as "clipped by the boundary you configured", which is the mental model that let the
option ship dead. Remove `hideWhenReferenceClipped`; **no alias, no deprecation** — 17 mechanical
renames and zero consumers (`npm view v-teleport-to` → 404).

Gate on `opts.hideWhenReferenceHidden !== false`.

`TeleportToEventDetail` gains `referenceHidden: boolean` and `hidden: boolean`, so a consumer who
wants the host *unmounted* rather than invisible can drive `v-if`. `useTeleportTo` gains a matching
`hidden` ref. (Note TT-7 already added `oppositeSpace` and `fit` to that payload — extend, don't
replace.)

Promote the `describe('hideWhenReferenceClipped option')` block at `vTeleportTo.test.ts:4104` to top
level while renaming; it is currently nested inside `describe('scrollContainer option')`, with which
it has nothing to do.

**Tests:** `mountDirective(el, { to: refOffscreen })` with **no other options** → hidden;
`{ hideWhenReferenceHidden: false }` → visible; `onPositioned` receives `referenceHidden: true`;
composable exposes it; `data-teleport-hidden` present/absent in step.

## Also in scope: the T4 window-listener floor (planner-approved)

`resolveScrollTargets` (`src/scroll-target.ts:19-35`) *replaces* the window listener when
`scrollContainer` names an element. Under the new default that means: narrow `scrollContainer` +
scroll the page → the reference leaves the viewport → **no recalc** → the host stays painted with
nothing to anchor to. The default would silently fail on a documented configuration.

Union `window` into the resolved target set whenever `hideWhenReferenceHidden !== false`, inside the
existing `diffScrollTargets` bookkeeping so `updated`/`unmounted` stay leak-free. Precedent: `resize`
is already unconditionally on `window` and exempt from `scrollContainer`. `{ scrollContainer: pane,
hideWhenReferenceHidden: false }` must attach **only** the pane — the opt-out is total.

Demo `04-boundary-scroll.vue` teaches the old trade-off in prose; that rewrite is **T7**, not this ticket.

## Acceptance

- TDD. Full suite green across 5 projects; **698 is the baseline**, not 646.
- `npx tsc --noEmit`, `npm run build`, `npm pack --dry-run` clean. From `playground/`:
  `pnpm typecheck`, `pnpm smoke`, `pnpm smoke:dist`.
- **Browser-verify** — this is a scroll-driven visibility feature and jsdom has no layout. Drive
  demo `01-basic-dropdown.vue` over CDP (`playground/scripts/lib/cdp.mjs`): open the dropdown,
  scroll the trigger out of the viewport, read back `visibility` and `data-teleport-hidden`; scroll
  back and confirm it returns. Then the T4 case: a card with a narrow `scrollContainer`, page
  scrolled, must still hide. **And confirm `v-show`-closed hosts stay closed throughout** — T0 fixed
  that and this ticket is the change that would have detonated it.
- Do **not** edit `playground/scripts/interactions/` — under migration. Note `pnpm geometry`
  currently fails for an unrelated reason (see PG-13); do not chase it.

T3 (split `overflow`), T5, T6 (docs) and T7 (playground) are separate tickets.
