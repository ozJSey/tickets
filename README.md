# tickets/

One file per dispatchable ticket. A ticket file is written to be handed to a subagent
**as-is**, with no context from the conversation that produced it — that is the point:
the session that wrote it may be gone.

- `BOARD.md` (repo root) — the live index: what is in flight, blocked, ready, done.
- `DESIGNS.md` (repo root) — archived design verdicts and their reasoning.
- `tickets/_STANDARDS.md` — definition of done for new packages.
- `tickets/<ID>-<slug>.md` — one dispatchable brief.

Every brief inherits the standing acceptance criteria in `BOARD.md`. Restate the
browser-verification requirement in the brief anyway; agents skip what is not in front of them.
