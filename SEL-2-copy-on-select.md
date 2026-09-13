# SEL-2 — `v-select-text`: copy the active selection

Owner: *"v-select-text, should allow user selection to be copied."* Full design and rejected
alternatives: `DESIGNS.md` → SEL-1. Standards: `tickets/_STANDARDS.md`.
Target **v2.1.0** — purely additive to the public surface.

## First: the README currently teaches a pattern that fails in two engines

Found 2026-09-06 when the owner asked *"if this function is supported where's the documentation
showing that."* It is not supported — but the README tells the reader to do it themselves, and the
advice is wrong:

- `README.md:324` — *"safe to hand straight to `navigator.clipboard.writeText`"*
- `README.md:347` — a worked example doing exactly that
- `src/types.ts:142` — the same in a JSDoc example

The package's **default** trigger is `'edge'`, which fires on mount. There is no user gesture, so
`writeText` is refused — **including in Chrome.** Measured by the independent audit 2026-09-06 in a
controlled matrix (one page, `clipboard-write` granted, `document.hasFocus() === true`):

| where the selection fired | result |
|---|---|
| default trigger (`edge`), on mount | **REJECTED — `NotAllowedError: Write permission denied`** |
| `trigger: 'click'` (real trusted click) | RESOLVED |
| `edge` flipped inside a real click handler | RESOLVED |

Running the `types.ts:142` snippet **verbatim** (no `.catch`) surfaces as an **uncaught page
exception**. This is not a Firefox/Safari caveat as originally assumed — the advice fails in the
engine its author tests in, and it is the first thing a new user hits.

**Fix these three sites as part of this ticket, whatever else lands.** Either qualify them with the
activation requirement and `trigger: 'click'`, or replace them with the new `copy` option once it
exists. Do not leave a recommendation the package's own default cannot honour.

## The surface

```ts
copy?: boolean   // default false
```
`boolean`, not a config object — there is nothing honest to put in one today (a text transform is
`v-copy`'s `source`; a feedback window is `v-copy`'s `feedback`), and `boolean` widens to a union
later without breaking anyone.

New exported types: `SelectTextCopyReason` = `'empty' | 'no-clipboard' | 'no-user-activation' |
'denied'`, and `SelectTextCopyDetail` = `{ ok, text, reason?, error?, selection }`.

**Copies `detail.text`. Never `getSelection().toString()`.** Three measured reasons, all already
pinned by the existing interaction spec: a `user-select: none` host stringifies to `""` and would
**wipe the user's clipboard** on a selection the package considers successful; Chrome mirrors a
focused field's own selection into the document selection but Firefox and Safari do not, so the
input path would copy the right thing in one engine and nothing in two; and engines insert their
own separators at block boundaries. Decisively, using it would mean `select-text` reports one
string while the clipboard receives another — the exact sin Run 22 found and fixed. Consequence:
composition with `match`, `start`/`end` and the whole-host path is **free**, no new code paths.

**An empty `detail.text` is never written** (`reason: 'empty'`), so `{ start: 5, end: 5 }` cannot
wipe a clipboard. No selection → no event → no copy attempt.

## The crux — user activation

`navigator.clipboard.writeText` needs transient activation. **`trigger: 'click'` is the only
universally-legal path.** The package's own default (`trigger: 'edge'`) fires on mount with no
gesture: Chrome tolerates it in a focused top-level document, **Firefox and Safari refuse**.

Therefore `copy` defaults to `false`, and a one-shot `console.warn` per element fires when `copy`
is set on a non-click trigger, naming `trigger: 'click'` as the fix — same shape as the existing
`user-select: none` diagnostic, which exists for exactly this "the directive appears not to have
fired" failure mode.

`navigator.userActivation?.isActive` is read **synchronously immediately before** the write and used
**only** to choose between `'no-user-activation'` and `'denied'`. It must not gate the attempt —
pre-blocking would make the package less capable than Chrome permits, and Safari does not expose
the API at all.

**No `execCommand` fallback.** The temp-textarea variant calls `.select()` and destroys the very
selection this directive exists to make; the no-textarea variant copies a different string.

## Output — never swallowed, never thrown

`select-text-copy` CustomEvent (bubbles, fires on **success and failure**, carrying `ok` / `reason`
/ the engine's `error`); `data-select-text-copy="pending|copied|error"`; and on the composable,
`copy(): Promise<SelectTextCopyDetail | null>` plus a **separate** `copyState` ref — do not widen
`state`'s union, the README's own example branches on it.

Not a throw (breaks the render from `mounted`/`updated`) and not a rejected promise on the
directive path (nobody holds it → unhandled-rejection noise in every consumer's console).

## Ordering — the invariant

New `src/copy.ts`, the **only** place a clipboard write happens, and **the write is always
initiated in the same synchronous turn as the selection it copies** — a write deferred past that
turn loses the activation that authorised it, which is the difference between working in Safari and
not. Add that to `ARCHITECTURE.md`.

The write is kicked off **before** `select-text` is dispatched, so a consumer's handler cannot
consume the activation first. Guarantees to document: what lands on the clipboard is exactly the
`detail.text` of the `select-text` that preceded it; `select-text-copy` never precedes its
`select-text`; one selection = at most one attempt = at most one event; a superseded attempt still
fires its own event (nothing swallowed) but does not write the attribute; an element unmounted
mid-flight dispatches nothing, and **the write still lands — say so in the README rather than
pretending otherwise.**

## Keyboard — in scope, and it is the blocker

`trigger: 'click'` has no `tabindex`, no `role`, no Enter/Space, so `copy` would ship **mouse-only**.
The portfolio rule from DZ-1 (`DESIGNS.md` → A11Y-1) is *prefer a real focusable control; inject
ARIA only when there is none* — and here there is none, so this **is** the case that needs the
`v-copy` treatment. Mirror `v-copy/src/events.ts`: injected `tabindex` + `role="button"` +
Enter/Space, only when `trigger: 'click'` is active, restored on teardown.

## Acceptance

- TDD. New `vSelectText.copy.test.ts` added to the `vue-3.5` and `vue-3.3` projects in
  `vitest.workspace.ts`; stub the clipboard as `v-copy/vCopy.test.ts:41-47` does.
- **jsdom proves nothing here** — probed: `navigator.clipboard`, `navigator.userActivation`,
  `isSecureContext` and `document.execCommand` are all `undefined`. Unit tests cover which string
  was offered, sync-start ordering, event ordering, the seq/unmount guards, the empty refusal and
  the reason mapping given a synthetic rejection. Nothing else.
- **Mutation-test the suite.** Three agents today found their own tests weaker than they looked;
  one caught a check passing for the wrong reason.
- **Browser-verify with trusted input.** `cdp.mjs:149` passes `userGesture: true`, which fakes the
  activation under test — so use `Input.dispatchMouseEvent` for the success case (as the COPY-4 and
  DZ-1a/d agents did) and `Runtime.evaluate` with `userGesture: false` for the refusal case. Grant
  `clipboardReadWrite` and read the value back. Prove: a real click copies the token; the clipboard
  holds the *rendered view*, not raw `textContent`; a deferred copy is refused and the event says
  why; **a refused copy leaves a pre-seeded clipboard intact**; keyboard Enter on the focused host
  copies.
- Two playground cards (12 → 14): click-to-copy tokens, and **the trap demonstrated on purpose** —
  flip the flag inside a handler (works) versus from `setTimeout`/rAF (refused, reason shown).
  That turns the constraint into something teachable instead of a README footnote.
- README: a `## Copying the selection` section with a trigger × engine matrix where **every cell is
  marked verified or unverified**. CDP is Chrome-only; Firefox and Safari rows are a manual pass or
  they ship unverified. Do not guess them.
