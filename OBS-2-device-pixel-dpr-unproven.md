# OBS-2 — `box: 'device-pixel'` is fixed but its headline case is UNPROVEN

Opened 2026-09-13 during the `v-observe` audit close-out (0.2.0). Not a suspected defect — a gap in
what this harness can witness. Recorded because "we could not check it" must never be filed under
"it works".

## What was wrong, and what is now proven

`resize` called `observer.observe(el)` with no second argument, so the observed box was always the
content box whatever `box` said. That matters because the observed box decides **when a callback
fires**, and `box: 'device-pixel'` exists for exactly one job: re-rastering a canvas when
`devicePixelRatio` changes on a zoom or a monitor move. A DPR change produces no content-box change,
so no callback ever arrived. README recipe 11 is that case verbatim.

Fixed in 0.2.0. Two browser checks stand behind it
(`playground/scripts/interactions/v-observe.mjs`):

- `ResizeObserver.prototype.observe` is instrumented via
  `Page.addScriptToEvaluateOnNewDocument`, and the recorded calls show `border-box` at mount,
  `content-box` and `device-pixel-content-box` after the demo's dropdown changes — so the value
  reaches the real API, and Chrome accepts it rather than throwing.
- With `box-sizing: content-box` and a fixed width, a padding-only change moves the border box and
  leaves the content box exactly where it was. `box: 'border'` reports it and `box: 'content'` does
  not, which is only possible if the box reaches `observe()`. Negative-controlled: reverting to
  `observe(el)` turns it red.

## What is NOT proven

**That a `devicePixelRatio` change on its own delivers a callback.** Attempted with
`Emulation.setDeviceMetricsOverride { width: 0, height: 0, deviceScaleFactor: 3 }`. It does move
`window.devicePixelRatio` (1 → 3, read back from the page), and nothing else happens: no
`ResizeObserver` callback, and `devicePixelContentBoxSize` stays at its old value.

The control that makes this a harness limit rather than a library one: a **bare** `ResizeObserver`
observing `device-pixel-content-box`, installed by hand in the page with no `v-observe` involved,
is equally silent under the same override. Evidence recorded in that run:

```
bare RO, before override: [{ dpcb: [248, 58], cb: [248, 58], dpr: 1 }]
bare RO, after  override: [{ dpcb: [248, 58], cb: [248, 58], dpr: 1 }]   ← no second entry
page devicePixelRatio after override: 3
```

Headless Chrome does not re-raster for the override, so the harness cannot tell a working
implementation from a broken one here. A check that cannot fail for the right reason is worse than
no check, so the DPR check was removed rather than left green.

## What would prove it

In rough order of cost:

1. **Real browser zoom.** `Ctrl` + `+` through `Input.dispatchKeyEvent` changes the page zoom, which
   changes `devicePixelRatio` and does re-raster. Needs a headed Chrome, and the key handling has to
   reach the browser rather than the page (`Browser.setWindowBounds` / a `--force-device-scale-factor`
   restart are the fallbacks).
2. **Two Chromes.** Launch a second instance with `--force-device-scale-factor=2` and compare the
   card's reading against the DPR-1 run. That proves the *read* scales, not that a *change* is
   delivered — weaker, but cheap, and it would have caught nothing that is already covered.
3. **Accept it as unprovable here** and say so in the README next to recipe 11, rather than letting
   a reader assume the check exists.

Whichever route: negative-control it. Revert `observeBox` to `observer.observe(el)` and confirm the
new check goes red, or it is not a check.

Do not use port 5199 for the run (`cdp.mjs` has no send timeout — PG-21).
