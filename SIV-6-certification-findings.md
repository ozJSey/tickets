# SIV-6 — four defects found by blind certification of `v-scroll-into-view` 1.3.0 (live on npm)

Found 2026-09-14 by an agent given the audit method and forbidden from reading `tickets/SIV-*`,
`tickets/quality/` or `DESIGNS.md`.

**Published at 1.3.0.** The certifier unpacked the published tarball and diffed it against a fresh
`tsup` build of current source: **byte-identical**. So there is no dist-staleness gap — every defect
below is what consumers run today, and the remedy is a **1.3.1 patch**.

**Verdict: partly works.** The default path is sound and the container path's arithmetic is
genuinely excellent — better than the author's own sweep proves. One live defect on the container
path (with a regression component) and one silent API trap.

## The bar it was held to

~2,400 rows of the certifier's **own** parity sweeps against `Element.scrollIntoView()` on twin
panes — geometry it constructed, not card 15's:

| Sweep | Rows | Result |
|---|---|---|
| border × padding × `scroll-padding` × CSS `scroll-margin` × size × `block`×`inline` (all 16) × approach | 1024 | **0 divergences** |
| RTL pane + RTL target, with scroll-padding, scroll-margin, offsets both axes | 768 | **0 divergences** |
| `offset` vs `scroll-margin` twin, container **and** native path | 256 | **0 divergences** |
| **direction combinations (rtl/rtl, ltr/rtl, rtl/ltr)** | 384 | **256 divergences** ← finding 1 |
| nested scrollers, `overflow:hidden` inner, `transform: scale` 0.5/1/1.75, scroll-snap | 56 | **0 divergences** |

---

## 1. HIGH — a target whose `direction` differs from its container breaks the horizontal axis

**Verified in source before filing.**

`src/geometry.ts:151` reads the flag off the **target**:

```ts
rtl: targetStyle.direction === 'rtl',
```

`src/execute-scroll.ts:86-88` then uses that same flag to pick the sign of the scroll-range clamp:

```ts
left: geo.rtl
  ? clamp(left ?? container.scrollLeft, -maxLeft, 0)
  : clamp(left ?? container.scrollLeft, 0, maxLeft),
```

The logical `start`/`end` flip **is** correctly a property of the target — native agrees, and the
certifier measured that. **The sign of `scrollLeft` is a property of the container.** Mixing them is
the bug.

- LTR pane + `dir="rtl"` target: **128/128 rows wrong.** Every horizontal alignment collapses to
  `scrollLeft 0` (native: 540 / 670 / 800).
- RTL pane + LTR target: **128/128 rows wrong** (lib 0, native −900).
- Matched direction: 0/128 wrong.

**Realistic repro — and it is this library's own headline use case.** A horizontally scrollable LTR
card rail whose items use `dir="auto"` for user-generated text. Chrome resolves `dir="auto"` +
Arabic to `direction: rtl`:

```
card 2 "مرحبا"  lib 0  native 223   Δ −223
card 4 "سلام"   lib 0  native 559   Δ −559
card 8 "عالم"   lib 0  native 1231  Δ −1231      (the 7 LTR cards: Δ 0)
```

The rail snaps to the start and the selected card is off-screen. Silent — no warning.

### The regression half

Note `left ?? container.scrollLeft`. With `inline: 'nearest'` (**the default**) and the target
already horizontally visible, `left` is `null` — and the clamp still rewrites `scrollLeft`. So a
**vertical-only** scroll destroys the pane's horizontal position.

A/B against published 1.2.0 (`PLAYGROUND_UNALIAS=v-scroll-into-view`), identical markup, LTR pane
pre-scrolled to 650, `dir="rtl"` target, `block: 'start'`:

```
1.2.0   scrollLeft 650 → 650    (native 650)  ✓
1.3.0   scrollLeft 650 → 0      (native 650)  ✗  −650px jump
```

Reproduces against `dist`, i.e. the published artifact. The CHANGELOG line that introduced it is
under **Added**: *"resolves `inline: 'start'` against the target's computed `direction`, so RTL lists
align the right edge and negative `scrollLeft` is handled."* Half right.

Scope is bounded — needs `container` set **and** the container horizontally scrollable **and** mixed
direction — but within it, 100%.

**Fix:** take the clamp sign from the **container's** `direction`; keep the logical `start`/`end`
flip on the target's. They are two different facts and need two different reads. Consider whether
`left === null` should skip the horizontal clamp entirely rather than re-clamping a value nobody
asked to change.

## 2. MEDIUM — `container: paneRef.value ?? undefined` silently falls back to native

The binding value is computed during render, before the parent's template ref is assigned, so the
**mount-time** scroll sees `container: undefined` → native path → every ancestor scrolls, including
the page. That is exactly what `container` exists to prevent, and there is no warning, because
`undefined` is a legal "no container".

```
O1  container: paneRef.value ?? undefined, condition:true on mount
      → pane 900 ✓ but the page-level scroller moved 601px   ✗
O4  same, v-for active row on mount → outer scroller moved 500px  ✗
O3  container: () => paneRef.value  (getter form)  → outer 0px    ✓
O2  container: paneRef.value  (raw null) → warns, scrolls nothing, no native fallback  ✓
```

**The type makes it worse.** `ContainerRef = HTMLElement | string | (() => HTMLElement | null)` and
`container?: ContainerRef` — `null` is not in the union, so TypeScript **forces** the `?? undefined`
spelling, the one that silently falls back. The spelling it rejects (`null`) is the one that honours
the README's promise that *"container always wins when set… there is no fallback"*. Confirmed with
`tsc --strict` against the published `.d.ts`.

The README lists the `HTMLElement` form **first with no caveat**, and cards 11, 14, 15 use it —
escaping only because they fire after mount.

**Fix direction:** admit `null` into the type so the honest spelling type-checks, and/or warn when a
`container` **key is present** but resolves to nothing, rather than treating it as "no container".
The getter form should be what the README leads with.

## 3. LOW — the one-shot warning latch disarms itself on the first false alarm

`warnOnce` latches globally per message per session, so an identical misconfiguration on a second,
unrelated element produces nothing (verified). The package's own tab demonstrates the cost: card 05
(`always`, chat tail) spends *"container has no scrollable overflow"* **at page load** on a perfectly
correct empty-chat demo — after which `container: 'body'` and `overflow: clip` containers are both
silent. Card 09's prose (*"open the console and each broken `container` has said so once"*) is only
true in a session where nothing tripped first.

Also: that message is misleading for `container: 'body'` — body *does* overflow; the document
scroller is `documentElement`, and `container: 'html'` works.

**Fix direction:** latch per element+message rather than globally, and stop the empty-chat demo
tripping it at load.

## 4. LOW — card 06's `inline` control is inert at desktop width (demo, not library)

`#grid-pane` is 1024px wide in a 937px scrollport, so `maxLeft` is 87. `inline: center | end |
nearest` all land on `scrollLeft 0` and are indistinguishable; only `start` moves, and only by
clamping to max. At 375px the same control works correctly (0 / 136 / 273). The blurb promises
*"every native alignment including the horizontal axis"*.

Separately, the directive is bound to the 23px 🎯 `<span>`, not the highlighted 128×80 cell, so
`block: 'start'`/`'end'` visibly clip the thing the eye reads as the target.

## What the repair got right

A/B against 1.2.0 confirms the headline fixes are real. 10px border + 30px CSS `scroll-margin`:

```
block:    start  center  end  nearest
1.2.0 Δ:    +40    +25   +10    +10
1.3.0 Δ:      0      0     0      0
```

Every other CHANGELOG "Fixed" entry checked out independently: `nearest` oversized-from-below,
`offset` on `center`/`end`, edge re-arm, the nested-scroller chain, the in-flight `scrollend`
destination, reduced-motion at scroll time, the composable's narrowed option type.

Also clean: reduced-motion (default→`instant`, explicit `smooth` still animates), `behavior: 'auto'`
deferring to CSS `scroll-behavior`, held-arrow-key at 60ms, hidden / `display:contents` / detached
targets, edge re-arm behind `v-if`, `always`, unmount cancellation, the full composable surface,
`:scope` vs plain selector in a `v-for` of panes, mobile 375px, packaging (10 files, no leakage), and
`tsc --strict` against the published `.d.ts`.

## Coverage gap this exposes — fix alongside

320 jsdom tests cannot see any of this. The repo's `interactions` spec for this tab is **12 checks
over 4 of 15 cards** (10, 11, 12, 15) — no RTL, no mixed direction, no horizontal axis, no nested
scrollers, no composable, no template-ref container form. Card 15's 192-row sweep is vertical-only,
with `inline` pinned to `'nearest'` and equal-width panes.

Note also: `pnpm docs:check` reports *"17/17 runnable samples **compiled**"*, which is not *run*. The
certifier ran them.

**A fix for finding 1 without an RTL/mixed-direction row in the interactions spec will regress
again.**

## Suggested release — `1.3.1`

Finding 1 fixed (clamp sign from the container, logical flip from the target), plus a README caveat
and a card-03 note for finding 2. Do not ship a new feature on top of 1.3.0 until finding 1 is out:
it is the only case where this package moves a scroller somewhere the browser would not, silently,
and the horizontal-reset half is **strictly worse than the version it replaced**.
