# DZ-1b/c — make the picker input focusable and named  (v-dropzone)

**Prerequisite for the default flip (DZ-1a). Changes no default.** Lands first so `main`
never carries a default-on click target keyboard users cannot reach.

**Why:** `ensurePickerInput` (`src/picker.ts:31-53`) creates a real `<input type="file">` and
hides it with `input.style.display = 'none'` (`:39`) — which removes it from the tab order.
HTML5 drag-and-drop has no keyboard or AT story by design, so this input is not a fallback,
it is the *only* accessible route to the feature.

## DZ-1b
Replace `:39` with a visually-hidden constant in `src/constants.ts`:
```
position:absolute; left:0; bottom:0; width:1px; height:1px;
padding:0; margin:0; border:0; overflow:hidden;
clip:rect(0 0 0 0); clip-path:inset(50%); white-space:nowrap;
```
`left:0; bottom:0` anchors the 1px box so it cannot create a scrollbar whatever the nearest
positioned ancestor is — browser-only claim, verify it there.

Add `pickerLabel?: string` to `src/types.ts`, written as `aria-label` from `syncPickerAttrs`
(`src/picker.ts:17-24`). Default `'Choose files'`, `'Choose file'` when `multiple: false`.

**Update two existing tests** that assert the old mechanism: `vDropzone.test.ts:974-978` and
`:3266` (both `expect(input.style.display).toBe('none')`).

## DZ-1c — focusability follows the affordance
`teardownClickToPick` (`src/picker.ts:74-81`) keeps the input alive for `api.open()`. Once
focusable that is a phantom tab stop on a zone the consumer opted out of.

| State | `tabindex` | `aria-hidden` |
|---|---|---|
| `clickToPick` on | none (naturally focusable) | no — `aria-label` set |
| `clickToPick: false`, `api.open()` only | `-1` | `true` |
| `enabled: false` | input destroyed | — |

Refactor both behind `applyPickerA11y(input, opts, { keyboardAffordance })` in `picker.ts`.

**Do not** touch `directive.ts:70` or `:116-117` (the default flip and its resolver are
DZ-1a). **Do not** add `role` or `tabindex` to the **host** — the host is never touched.

**Acceptance:** 264 existing tests green plus new ones; in particular the 9 interactive-guard
tests at `:995-1094`, the reuse guards at `:3314`/`:3348`/`:3351`, and the toggle at `:1223`
must be untouched and passing. `npm run build`, `npm pack --dry-run` clean. **From
`playground/`: `pnpm typecheck` clean** — a previous agent broke exactly that command with a
type that did not satisfy Vue's `CSSProperties`.

**Browser-verify:** Tab from a preceding element lands on the zone's `input[type=file]`;
Enter reaches `input.click()` (count it by patching `HTMLInputElement.prototype.click` for
`this.type === 'file'` and restoring in a `finally` — the pattern at
`scripts/interactions/v-dropzone.mjs:215-227`; the native dialog never opens); the input
contributes zero layout (snapshot host and document `scrollWidth`/`scrollHeight` before and
after); the accessible name is right (`Accessibility.getPartialAXTree`); the host has no
injected `role`/`tabindex`, including on a host that already has its own `role`.

**Do not edit `scripts/interactions/v-dropzone.mjs`** — the harness is under re-evaluation.
Check `:248` (`style.display === 'none'`) will break; report it, do not fix it.
