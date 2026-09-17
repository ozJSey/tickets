# OBS-1 — `each` delegated observation, and the three audit P1s

**Owner, 2026-09-16 (audit walkthrough, stop 10):** build all three ideation ideas. This ticket
takes the **mutation path** — `each` plus the correctness P1 that lives in the same module — so it
cannot collide with OBS-2 (intersection path). One agent.

## A. P1, correctness — `children:added` and `children:removed` are not distinguished

`src/mutate-records.ts:38`. Subscribing to one delivers both. A consumer asking only for removals
gets callbacks on every insertion — silently wrong, and it makes the event names a lie. Fix first,
before `each` builds on this module, and pin with a test that fails against current code
(subscribe to `added` only, remove a child, assert **no** callback).

## B. `each` — observe matching descendants with one observer

```html
<article
  v-html="renderedMarkdown"
  v-observe="{ intersect: { each: 'img[data-src]', once: true, on: (e) => lazyLoad(e.target) } }"
/>
```

- **The case nothing else can serve:** Vue directives cannot exist inside `v-html`, markdown output
  or CMS content, so today nobody can lazy-load or animate-in elements the framework never
  rendered. `each` observes them by selector from the container.
- **One `IntersectionObserver` for all matches**, not one per element — this also collapses 500
  directive instances on a long `v-for` into a single observer.
- An internal `MutationObserver` auto-observes matching children as they are added and unobserves
  them as they leave. That coupling is the whole point: only a directive that owns **both**
  observers can do this, which is why VueUse structurally cannot ship it as three wrappers.
- Works with the existing `intersect` options (`once`, thresholds, root margin) unchanged — `each`
  changes *what* is observed, not *how*.
- Teardown must unobserve every element it ever observed and disconnect the internal MO; a leak
  here is the same class as the `generatedIds` P1 found in v-keyboard-navigation.

## C. P1 — dead `repository`/`homepage`/`bugs` on published 0.2.0

`package.json:7` points at `github.com/ozJSey/vue-observe`, which 404s anonymously. Local was
corrected in `19d2c7d`; the **published artifact stays wrong until the next publish**. Confirm the
local URL resolves anonymously; invent nothing (META-1 is the owner's call). Note in the completion
report that the npm page only clears on the next release.

## D. P1 — CHANGELOG claims 0.1.0 "was never published"

`CHANGELOG.md:16`. The registry lists **both** 0.1.0 and 0.2.0. Corrected here rather than in
DOC-1 because this package needs its own patch anyway for A. Coordinate: DOC-1 owns the twelve-package
sweep and the publish-time gate; this ticket owns v-observe's instance.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. A's failing-first test, plus a browser check that added/removed fire independently.
2. **Real-browser check for `each`** on a `v-html` card — the case the feature exists for — plus a
   dynamically-appended child auto-observed after mount. Read results out of the live DOM.
3. **Negative control:** without `each`, the same `v-html` content is never observed (proving the
   check exercises the feature, not ambient behaviour).
4. Observer-count assertion: N matching elements produce **one** observer, not N.
5. Teardown: unbind, assert the MO is disconnected and nothing is still observed.
6. Release: **minor** (`each` is a real capability); the major position does not move.
