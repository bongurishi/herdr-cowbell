# herdr-cowbell

Herdr TUI plugin: agent status icons in tab labels. Start with [PLAN.md](PLAN.md). It has the design, phases and open questions, and the facts about herdr that the design depends on.

## Rules

- **Never run `herdr …` against the live server while developing.** This repo is usually opened from inside a herdr pane, where `HERDR_SOCKET_PATH` points at the user's real session. Use the sandbox procedure in PLAN.md → "Testing". Before linking, confirm `herdr plugin list` in the sandbox shows no plugins.
- Link or install into the real herdr only when the user explicitly asks.
- Node ≥ 18, no npm dependencies, no build step. Tests use `node --test`.
- Keep herdr socket calls in `lib/socket.js`. Keep label/icon logic pure and covered by unit tests.
