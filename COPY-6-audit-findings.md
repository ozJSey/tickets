# COPY-6 — `v-copy` audit findings (library-level)

From the independent audit, 2026-09-06. **The library passed its default path and every README
recipe** — run in a real Vite + Vue 3.5.41 app against both source and `dist`, with every copy
confirmed by reading the real clipboard back against a primed sentinel. These are the real defects
found around the edges. The three broken cards are **PG-14**, not this ticket.

**F5 — MEDIUM-HIGH, default path.** `dedupe` is on by default and compares **text alone**, so two
differently-labelled bindings sharing one rich sink collapse to a single row and the older label is
silently lost: `v-copy:colA.rich="log"` and `v-copy:colB.rich="log"` over the same payload leave
`[{text:"same@value.dev", key:"colB"}]`. The README's multi-copy recipe says many bindings share one
history and "the argument labels entries" — and the `dedupe` section never says labels do not
participate in the comparison. *(This was a deliberate design call in COPY-1 — "`key` is ignored by
the comparator, the latest wins" — but the README does not carry it, and default-on makes it the
common case. Decide: document it, or make `key` participate, or scope dedupe per key.)*

**F6 — MEDIUM, default path, silent.** An `undefined` or empty binding fails cheerfully:
- `v-copy="maybeToken"` with `maybeToken === undefined` copies the **element's visible label**
  instead — a real trap for async-loaded tokens.
- `v-copy="emptyRef"` writes `""` to the clipboard, returns `success: true`, sets `[data-copied]`
  and announces "Copied". **The user's clipboard is wiped while the UI says it worked.**

A `warnOnce` channel exists and fires correctly for `key` on a plain sink; neither of these uses it.
The README says a primitive `ref` "cannot be a history target" without saying what happens instead.
Refusing an empty write matches the rule already agreed for `v-select-text` (SEL-2's `'empty'` reason).

**F7 — LOW, doc contradiction.** `.once` is documented as "copy at most once, then detach", but
`ctrl.copy()` still copies after it has fired. The latch is in `events.ts:onTrigger`;
`controller.ts` calls `executeCopy` directly. Decide which is right and make the docs match.

**F8 — LOW.** A custom `trigger` **adds to** activation rather than replacing it:
`{ trigger: 'dblclick' }` on a `<span>` still gets `tabindex=0` + `role="button"` and copies on
Enter/Space. Undocumented second path, and `role="button"` misdescribes a double-click-only
affordance to assistive tech.

**F9 — LOW, demo prose.** Card 03's manifest blurb says "every copy lands in it", which default-on
`dedupe` makes false; the card body never mentions dedupe despite being the history card, and its
tags advertise `max`, which the card does not expose. Card 08's `.prevent` counter is labelled
"navigation attempts" but counts clicks.

**F10 — publish blocker, factual.** `package.json` still declares `"name": "v-copy"`. The registry
has `v-copy@0.1.0` owned by someone else and `@ozjsey/v-copy` does not exist, so **`npm publish`
cannot succeed as it stands.** The README (`npm install v-copy`, every `from 'v-copy'`), the
`repository`/`homepage` URLs, and the missing `publishConfig.access` all still point at the
unscoped name. Covered by PUB-1.

## Acceptance

- F5 and F6 resolved with a recorded decision, tests, and README changes that match behaviour.
- Do not regress what the audit confirmed working: bare binding, feedback timing, `tabindex`/`role`/
  Enter/Space, the shared `aria-live` region, the `dedupe` worked example character-for-character,
  all four compare modes, every modifier, the controller surface incl. driver unmount/remount, the
  `execCommand` fallback path, plugin defaults, and card 12's exact TSV.
- Re-verify in a real app, not only the playground, until **PG-14** is fixed.
