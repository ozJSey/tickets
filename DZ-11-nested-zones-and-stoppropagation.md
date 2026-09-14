# DZ-11 — all four drag listeners `stopPropagation()` unconditionally

P2. Quality-audit finding 12. **Documented in 0.1.1** (README Behavior list + a module docblock in
`drag.ts`); the behaviour itself is unchanged, because changing it is a semantic change to every
zone on a page and does not belong in a patch.

`drag.ts` — `dragenter`, `dragover`, `dragleave` and `drop` each call `event.stopPropagation()`.
Two consequences, both real and both previously undocumented in 718 lines of README including a
26-item Behavior list:

1. Any ancestor's `@dragover` / `@drop` handler never fires while the pointer is over a zone, so a
   page-level "drop anywhere" overlay goes dead over every dropzone on the page.
2. **A `v-dropzone` nested inside another one does not work.** The outer zone's depth counter needs
   the `dragenter` bubbling up from the inner zone to stay balanced; without it the outer zone's
   `active` styling drops off the moment the pointer crosses into the inner one.

## The call

- Keep it and treat nesting as unsupported (documented, which is now true), **or**
- Drop `stopPropagation` from `dragenter` / `dragleave` only — the two the counter depends on —
  and keep it on `drop` so a file is not handled twice. That fixes nesting and the overlay's
  hover feedback, and leaves the "one drop, one handler" guarantee intact. Needs measuring: an
  outer zone would then also flip to `active` when the pointer is over an inner zone, which may be
  what a consumer wants (it is one drop target visually) or may not.
- `preventDefault` on `dragover` must stay unconditional either way — without it `drop` never
  fires at all, which is the canonical drop-zone bug this package exists to fix.

## Acceptance

- A playground card with a nested zone, because this is exactly the kind of thing jsdom's
  synthetic events model badly — the depth counter has to be watched in a real browser.
- A page-level overlay card, or at minimum a browser check that an ancestor listener does or does
  not fire, matching whatever the decision is.
- Whatever changes ships as a minor with a `CHANGELOG.md` **Changed** entry, not a patch.
