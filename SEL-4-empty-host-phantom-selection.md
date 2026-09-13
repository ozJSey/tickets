# SEL-4 — bare binding on a not-yet-populated host fires a phantom event and wipes the page selection

Found by the independent audit, 2026-09-06. **Default path, no options.** Highest-impact runtime
finding in `v-select-text`.

## Reproduce

```vue
<p v-select-text>{{ fromApi }}</p>   <!-- text arrives after mount -->
```

Observed in Chrome, single host, fresh load:
```
at mount:  text === ''  →  find-range.ts returns mode:'all'  →  a COLLAPSED range is installed
           selection: ""   rangeCount: 1   collapsed: true
           event fired: {"start":0,"end":0,"text":"","direction":"forward","kind":"text"}
then:      hostText = "Text that arrived from the API after mount."
           → nothing is ever selected. `updated` sees prev === true, so the 'edge' trigger
             never re-fires.
```
Same for a whitespace-only host.

## Three defects in one

1. **A phantom event.** `README.md:328` states *"No selection means no event."* The library fires one
   claiming a selection, with `text: ""`. A consumer following the README's own clipboard recipe
   then writes an empty string to the clipboard.
2. **It destroys the user's selection.** `src/selection.ts:124` calls `sel.removeAllRanges()` before
   installing its own. A host that renders empty therefore **wipes whatever the user — or another
   host — had selected**, and installs a collapsed range in its place. This is a directive with no
   options, on a host with no content, reaching out and clearing global state.
3. **It never recovers.** The `'edge'` trigger has already fired, so the text arriving changes
   nothing. The consumer's only escape is `trigger: 'always'` or a manual `enabled` cycle, neither
   of which the README connects to this situation.

## What it should do

An empty resolved text should **select nothing, fire nothing, and leave the document selection
untouched** — and the edge should not be considered spent, so the selection happens when content
arrives. Confirm that reading against `find-range.ts`'s `mode:'all'` path and the `'edge'`
bookkeeping in `src/state.ts` before changing behaviour; there may be a reason the edge latches
that is not visible from the outside.

Decide explicitly, and document: is an empty host a *no-op* (nothing happened yet) or a *failure*
(warn)? A no-op that later fires when text arrives is the behaviour a consumer expects from
`{{ fromApi }}`.

## Related, same audit — `detail.text` can describe a selection that does not exist

All read back as `getSelection().toString() === ""` while the event reports full text. Only one of
four warns:
- host itself `display: none` → reports the text, selects nothing, **no diagnostic**
- host inside a `v-show="false"` ancestor (a tab panel or modal that mounts hidden) → reports
  `{"start":0,"end":49,...}`, paints nothing. It recovers if shown later *and* nothing else touched
  the selection — but the mount fire is the only fire.
- a range covering exactly a synthetic separator (`<li>abc</li><li>def</li>` with `{start:3,end:4}`,
  or `one<br>two`) → collapsed range, `detail.text: " "`
- `user-select: none` → **does** warn (verified once-per-element)

Make the reported payload agree with what is actually selected, or warn. Silently reporting a
selection that was never painted is the same class of defect as the phantom event.

## Acceptance

- TDD; baseline 452 tests across 3 workspaces.
- **Browser-verified**: the `{{ fromApi }}` case selects the text when it arrives; a pre-existing
  user selection survives an empty host mounting; no event fires for an empty resolution.
- `README.md:328`'s "no selection means no event" becomes true, or the sentence changes.
- Do not regress the audit's confirmed-working surface: whitespace/text-map claims, `match` /
  `matchIndex` edges, `direction`, all four triggers, inputs, contenteditable, `useSelectText`,
  and the 23/23 existing interaction checks.
