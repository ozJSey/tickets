# DZ-5 — two README recipes are copy-paste broken on the headline upload paths

Found by the **first audit** and again by the **blind re-audit** — unfixed in between because DZ-2
covered the paste conflict and nobody carried these forward. A tracking failure, now its own ticket.

**Recipe 3 sends `Authorization: Bearer undefined`.** Pasted verbatim, intercepting
`XMLHttpRequest.setRequestHeader`, the header is literally `"Bearer undefined"`. A setup ref is
already unwrapped inside a template expression, so `token.value` is a double-unwrap. The upload
still succeeds against the mock server — **on a real one it is a 401 the consumer will blame on
their own auth wiring.**

**Recipe 4 (S3 presigned / custom transport) throws on every upload.** `onError` fires
`_ctx.fetch is not a function`, `data-dropzone="error"`, zero uploads. Confirmed against
`vue/compiler-sfc` directly: `fetch` is not in Vue's template-globals allowlist, so it compiles to
`_ctx.fetch(...)` in **any** Vue 3 SFC.

**Same root cause, and the caveat is wrong:** the README says *"Every other option is safe to write
inline; only `ref` carries the ref itself."* False. **Any** non-allowlisted global — `fetch`,
`FormData`, `localStorage`, `AbortController` — and any `.value` breaks inside a template
expression. Rewrite the caveat to say what actually holds, and move both recipes' option objects
into `<script setup>`.

## Also — the `position: relative` write is a one-way ratchet

With a conditional position, e.g. `@media (min-width: 900px) { .dz { position: sticky; top: 0 } }`:

| | computed | inline | stuck |
|---|---|---|---|
| mounted wide (rule live) | `sticky` | — | ✓ directive stands down |
| resized narrow, no re-render | `static` | — | **anchor absent — the Tab-scroll jump is live in this window** |
| after any re-render | `relative` | `position: relative;` | — |
| resized wide again | `relative` | `position: relative;` | **`sticky` permanently dead** |

The README's *"Set any position of your own and the directive leaves the host alone"* holds only for
a `position` that is unconditional at mount. Inline beats any stylesheet and nothing removes it.
Decide: re-evaluate on resize, use a non-inline mechanism, or document the limit precisely.

## Doc nit

The README prescribes `.dz:focus-within` for the focus ring; that also lights for a zone's own inner
button. The demos use `:has(> input[type='file']:focus-visible)`, which the README never mentions.

## Acceptance

- Both recipes run when pasted into a real Vue 3 SFC. **Verify by pasting, not by reading.**
- The inline-expression caveat states the real rule.
- A decision recorded on the ratchet.
- Do not regress: the re-audit passed 61 checks across 13 cards, 7 host shapes, mobile width and
  `dist`, and re-verified DZ-3 with its own negative control.
