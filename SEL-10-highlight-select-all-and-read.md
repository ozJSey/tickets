# SEL-10 — v-select-text: `highlight` mode, `trigger: 'select-all'`, and `read()`

**Owner, 2026-09-16 (audit walkthrough, stop 12):** all three approved.

**Prerequisite — this ticket is dispatched only after v-select-text's line-by-line audit lands.**
That audit died on the session limit and is queued; all three features build on `text-map`, and
shipping features onto an un-audited module is exactly the sequencing mistake this walkthrough
exists to avoid. When the audit returns, fold its findings in before building.

All three share `text-map` and the selection pipeline, so: **one agent, one ticket.**

## A. `highlight` — paint every match, zero DOM mutation

```html
<article v-select-text="{ match: query, highlight: true }">
  The quick <strong>brown fox</strong> … every occurrence painted, across the
  <strong> boundary, none inside a v-show="false" panel.
</article>
```

```css
::highlight(select-text) { background: gold; }
```

- Uses the **CSS Custom Highlight API** (`Highlight` + `CSS.highlights`) — ranges are painted, the
  DOM is never touched. No `<mark>` injection, no hydration fights, no re-render destruction.
- Highlights **all** matches (string or `RegExp`), across nested elements, using the existing
  `match` and `whitespace` semantics — `text-map` already resolves the subtree to rendered text
  with per-character node anchors, which is the entire hard part and why this is small here.
- **Skips `display: none` subtrees** so a hidden tab panel produces no ghost hits.
- Named-group form (`highlight: 'lint'`) so two hosts can be styled differently.
- **Never touches `document.getSelection()`** — highlighting and selecting are independent; a user's
  own selection must survive.
- Feature-detect `CSS.highlights`; where absent, no-op with a dev warning rather than falling back
  to span injection.

## B. `trigger: 'select-all'` — scope Ctrl/Cmd+A to the host

```html
<pre v-select-text="{ trigger: 'select-all', copy: true }"><code>npm i @ozjsey/v-select-text</code></pre>
```

- Intercepts select-all **only while focus is inside the host**, selecting the host's rendered text
  instead of the document. Outside the host, the browser's behaviour is untouched.
- `copy: true` composes legally here because Ctrl+A is a **real user gesture** — one keystroke
  selects and copies. Call that out in the README; it is the reason this trigger is more than a nicety.
- Reuses the keyboard/focus affordance `trigger: 'click'` already establishes (tabindex, role
  semantics) — do not invent a second affordance model.

## C. `read()` — the inverse text-map

```ts
const sel = useSelectText({ target: () => article.value })
const mark = sel.read()          // { start: 128, end: 154, text: '…' } in rendered-text coords
sel.update({ start: mark.start, end: mark.end }); sel.select()   // restore, later, post-refactor
```

- Maps the user's live selection **back** into the same rendered-text coordinates the package
  already writes in — coordinates that survive re-renders, SSR/hydration and markup refactors
  because they index what the reader sees, not DOM node paths.
- Returns `null` when the selection is empty or falls outside the target.
- Partial-node and multi-node selections must round-trip exactly; that is the test that matters.
- Honest framing for the README: VueUse's `useTextSelection` also "reads the selection" — it hands
  back live DOM objects. The beat is **durable coordinates**, not the act of reading.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. **Real-browser checks throughout** — jsdom implements neither the Highlight API nor real
   selection geometry; anything it can prove is plumbing and must be reported as such.
2. A: matches painted across a nested-element boundary; a `display: none` panel produces none;
   the user's own selection is intact afterwards. **Negative control:** `highlight` off → `CSS.highlights`
   empty.
3. B: real Cmd/Ctrl+A via CDP with focus inside the host → only host text selected; focus outside →
   document behaviour unchanged. With `copy: true`, read the clipboard back.
4. C: round-trip property check — select a partial, multi-node range, `read()`, mutate the DOM
   around it (re-render), restore, assert identical text.
5. **Also in scope, from the known docs drift:** the README still documents `condition` while the
   source moved to `enabled` (deprecated alias) and added `trigger`/`match`/`copy`/contenteditable/
   `useSelectText`. Correct it in this pass — the package's docs have been stale for a while.
6. Release: **minor**; the major position does not move.
