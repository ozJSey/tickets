# Definition of done — new packages

Owner-stated, 2026-09-05. Applies to **`v-keyboard-navigation`** and the **write-behind
cache** package, and to any package added after them.

> "Always same standards, smoke tested, same architecture, vue first, typed, in playground
> AND well reviewed."
> "DX is a top priority as well with smart defaults."
> "Of course these 2 new libraries must be registered to our playground."

## Non-negotiable

1. **Vue first — with one settled exception.** Design Vue-first by default; do not return
   "ship it as neutral TypeScript" as a *recommendation*. The original wording forbade
   framework-agnostic-with-an-adapter outright and named the write-behind survey as the case
   it settled. **The owner overturned that for write-behind specifically on 2026-09-14:**
   *"Write behind doesn't really need to be Vue, make a typescript version of it as well, and
   most ideally vue package uses the package"* — then *"Net new project for sure tho."* So
   `@ozjsey/write-behind` is the engine and `@ozjsey/vue-write-behind` is an adapter over it,
   by instruction, and it is **not** a violation of this standard.
   The rule that survives: the split has to be earned. It was here because the state machine
   already had no Vue in it — three of six modules moved untouched, and their 81 test
   declarations imported no Vue at all. Do not propose an adapter split for a package whose
   core is entangled with reactivity; propose it when the core is already sitting there
   framework-free, or when the owner asks.
2. **Same architecture.** `src/` split into single-purpose modules behind a thin re-export
   entry, plus an `ARCHITECTURE.md` naming the module map and the invariant the split
   protects. Rationale (`CLAUDE.md`): most people copy the source rather than install it,
   so the readability of the files *is* the distribution.
3. **Typed.** Exported public types, no `@ts-ignore`, no casts, no defensive branches for
   states the types exclude.
4. **Every README links to the live site.** Owner, 2026-09-13. The canonical form is
   `https://ozjsey.github.io/npm-portfolio-playground/#<library-id>` — the hash is the demo folder
   name. A package exempt from having a tab links to the site index instead. This goes near the top
   of the README, above the install line: for a published package the README is the npm page, and
   the demo is the fastest way to understand what the package does.

5. **Registered in the playground, with documentation — unless it genuinely cannot be.**

   Owner, 2026-09-06: *"everything needs to be in playground and well smoke tested except for
   obvious ones like dependency grouper, it simply is a file editor library."*

   The exemption is for things a browser tab cannot honestly host, not for things that are
   inconvenient. **Exempt: `dependency-grouper`** (a CLI that rewrites `package.json` files on
   disk). An exempt project still owes a documentation view and a changelog, and its README must
   carry whatever a demo would have shown.

   **Not exempt: `bigdecimal-string`** — a REPL card showing the exact decimal string beside the
   IEEE-754 float answer makes its whole value proposition self-evident, and `TASKS.md` already
   names it the highest-value demo in the backlog.

   Everything else: A tab of its own, one card per
   feature, added in the same run as the code — not as a follow-up. Both new packages owe this.
   Since 2026-09-05 every project owes **two views: Playground and Documentation**, deployed to
   GitHub Pages and linked from that package's README. See `tickets/DOCS-1-pages-deploy.md`.
   Docs are *rendered from the package README*, never hand-authored twice — this repo has
   repeated, documented README-vs-source drift.
5. **A `CHANGELOG.md`, honest and sourceable.** Publishing without one is not allowed; a version
   number nobody can decode is not a release. An entry describing behaviour that has not been
   independently confirmed is worse than a missing entry. See `tickets/DOCS-2-changelogs.md`.
6. **The documentation is a smoke test too.** Owner, 2026-09-13: *"We can treat documentation as
   smoke test too."* The docs view is a rendered artifact like any card, so it is checked like one:
   it renders for every package, its code samples run when pasted, its links resolve, and any claim
   it makes is demonstrated somewhere a check can see. A README that documents a call which throws
   — which happened in `v-select-text` — is a failing gate, not a docs chore. See `tickets/DOCS-3`.

7. **Smoke tested, seriously.** Per the standing rule: a real-browser check that drives the
   behaviour and reads the result back out of the live DOM. No browser check -> reported
   UNPROVEN, never as passing. A red check beats a missing one.
7. **Well reviewed.** Adversarial review pass over the finished implementation before it is
   called done. Runs 19-22 all shipped "done" on green gates over broken behaviour; Run 22's
   reviewers found a P0 that hung the tab.
8. **TDD.** Failing test, minimum to pass, refactor.

## DX is a first-class requirement, not polish

**Smart defaults — the default is what you would have chosen.** The owner has now made this
call three times in one session, all in the same direction:

- `v-teleport-to`: hide when the reference scrolls away -> **default on**, opt out.
- `v-dropzone`: click opens the picker -> **default on**, opt out.
- `v-copy`: `dedupe` -> **default on**, opt out.

So for a new package: the bare binding must do the right thing with no options. Options
exist to *opt out* or to handle the unusual case. A feature that only works once you have
read the README and passed a flag has failed this bar.

Corollaries that fell out of today's work and apply to both new packages:

- **A default that makes something clickable must make it keyboard-operable** — preferring a
  real focusable control over injected ARIA (see `DESIGNS.md` -> DZ-1, which resolved A11Y-1).
- **A default-on behaviour must not silently fail on a documented configuration.** The
  `scrollContainer` hole in TT-3/T4 is the worked example.
- **Never render or report a state with no signal.** TT-13: a popover clamped to `maxHeight: 0`
  simply vanishes, and nothing tells the consumer why.
