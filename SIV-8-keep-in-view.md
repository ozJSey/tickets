# SIV-8 — `keep: true`: hold the target in view while layout shifts

**Owner, 2026-09-16 (audit walkthrough, stop 11): approved.**
**Runs AFTER SIV-7** — it consumes that ticket's settle signaling, and both touch the executor.
Dispatching them concurrently would have two agents editing the same module.

## The gap

The directive answers *"get me there once"*; nothing in the Vue directive space answers *"and keep
me there"*. Streaming AI chat, image loads, an accordion expanding above the active row — all shove
the target back out of view one frame after the scroll lands.

The platform's own fix does not reach: `overflow-anchor` is `auto | none` only (you cannot nominate
an anchor element), and it is absent from stable Safari.

## Design

```html
<!-- chat: pin the tail while streaming; scrolling up releases it -->
<div id="pane" style="overflow-y: auto">
  <div v-for="m in messages" :key="m.id">{{ m.text }}</div>
  <div v-scroll-into-view="{ keep: true, container: '#pane', block: 'end' }" />
</div>

<!-- or: keep the ACTIVE row visible while the accordion above it expands -->
<li v-scroll-into-view="{ condition: i === activeIndex, keep: true, container: '#list' }">
```

- `ResizeObserver` on the target **and its scroller chain**; re-run the existing executor on
  geometry change.
- **User-intent release** — this is what makes it usable, and what every naive stick-to-bottom gets
  wrong: a user scroll *away* from the target releases the hold; scrolling the target back into
  view re-arms it. Released state is reflected as
  `data-scroll-into-view-state="released"` so a "jump to latest" pill is pure CSS:

  ```css
  [data-scroll-into-view-state="released"] ~ .jump-pill { display: block }
  ```

- **Must not fight itself.** The executor's own scroll must never be mistaken for user intent, and
  a re-scroll must not trigger another re-scroll. Use SIV-7's settle signal to gate re-entry — this
  is the ticket's central risk and where a naive implementation produces an infinite scroll loop.
- Point it at a tail sentinel and it IS stick-to-bottom; point it at an arbitrary element and it is
  something no incumbent ships (vue-stick-to-bottom is bottom-only and component-shaped;
  use-stick-to-bottom is React and replaces native scrolling with a spring).

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **Real-browser checks** only for behaviour (jsdom has no layout): streaming content appended in
   a loop, target stays in view; an element above the target expands, target stays in view.
2. **Release/re-arm check**: drive a real user scroll away via CDP → assert `released` and that
   appended content no longer moves the pane; scroll back → assert re-armed.
3. **Loop safety**: assert a bounded number of scroll operations for N content changes — a test
   that would fail if the executor re-triggered itself. This is the named risk; it needs a check
   that can actually catch it.
4. **Negative control:** same card with `keep` absent — the target is shoved out of view, proving
   the checks exercise the feature.
5. `reduced-motion` behaviour stated and checked, since this repeats scrolls.
6. Release: **minor**; the major position does not move.
