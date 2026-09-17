# VC-1 — v-copy: app-wide history (plugin sink, native-copy capture, opt-in persistence)

**Owner, 2026-09-16 (audit walkthrough, stop 6):** build the plugin-level sink and `.listen`;
persistence only reshaped — *"I don't wanna clutter local storage without asked. We can do it
optionally. And we need them to choose the way to store it."* `.sensitive` is parked, not killed —
do not build it here, and design nothing that would block it later (history entries stay
structurally able to carry a masked/omitted flag).

One ticket, one agent: all three land in the same sink/history modules and README; three agents
would collide there.

## A. Plugin-level sink — one line, every bare `v-copy` records

`CopyPluginOptions` (src/types.ts:250 — today `{ max, dedupe, feedback, announce }`) gains
`sink` and `rich`. A bare `v-copy` with no per-binding sink records into the plugin's; per-binding
`sink` (or `false`) overrides. `.sensitive`-shaped entries are out of scope but the override
precedence must leave room for a binding to exclude itself.

```ts
app.use(VCopyPlugin, { sink: appClipboard, rich: true, max: 20 })
```

## B. `.listen` — native ⌘C / context-menu Copy lands in the same history

`v-copy.listen` on a container records every native copy inside it through the existing
dedupe/promote pipeline. Mechanism: the DOM `copy` event + `getSelection().toString()` (and the
existing input-field path in selection.ts) — what the browser is writing IS the selection at that
moment, so **no clipboard-read permission and no prompt**. Rich entries carry `key: 'native'` so
the picker can badge them.

**Named risk — no double-recording.** A copy the directive itself performs must not also be
captured by an enclosing `.listen`: `navigator.clipboard.writeText` fires no `copy` event, but the
`execCommand('copy')` fallback DOES. Guard with a directive-owned "this copy is ours" flag around
the fallback path, and pin it with a test on the fallback branch specifically.

## C. `persist` — opt-in, consumer-chosen storage

- **Default OFF. Nothing touches any storage unless the consumer passes `persist`.**
- Shape: `persist: { key: string, storage?: 'local' | 'session' | StorageLike }` where
  `StorageLike = { getItem, setItem, removeItem }` (sync). Default backend when only `key` is
  given: `'local'`. Async adapters (IndexedDB et al.) are a **non-goal** this pass — say so in
  the README rather than half-shipping them.
- Cross-tab merge (the part nothing on npm ships): merge by entry timestamp through the existing
  dedupe+promote semantics — never last-writer-wins array clobbering. Auto-merge applies to the
  `'local'` backend via the `storage` event; custom adapters get no auto-merge (documented).
- Interlock honoured even though `.sensitive` is parked: entries flagged omit-from-persistence
  (the structural room from the preamble) are never written.

## Acceptance — standing criteria (BOARD.md), agent at xhigh effort

1. Real-browser checks: A and B each get a playground card extension; B's check drives a **real
   trusted copy gesture** through the CDP input path against selected prose and reads the history
   back out of the live DOM. The fallback double-record guard gets its own negative control
   (remove the guard, watch the duplicate appear).
2. C: unit-level with a fake StorageLike + real-browser reload check on the `'local'` backend;
   negative control asserts **zero storage writes** when `persist` is absent — that is the
   owner's actual requirement, test it as such.
3. Two-tab merge is browser-checked (two iframes suffice); the clobber failure mode is the
   negative control (bypass the merge, watch one tab's history vanish).
4. README: three sections, each opening on its playground card per DOCS-4 conventions; persistence
   documented as opt-in first, mechanics second.
5. Version: **patch** (owner rule 2026-09-16). `.sensitive` explicitly listed under "parked, by
   owner decision" in the ticket-completion note, not silently dropped.
