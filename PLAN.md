# herdr-cowbell — implementation plan

Status: **planning, nothing implemented yet.** Written 2026-10-29 from herdr TUI
0.9.3 (installed) and the herdr source/docs at 0.9.5. Re-check the facts marked
*verify* before relying on them.

## Goal

Show each tab's agent state in the herdr TUI tab bar, the way the sidebar does
for agents:

```
 promptator │ ◐ grundza 1 │ ? grundza 2 │ ✓ cervenak │ grundza 3 │ term
```

- `◐` an agent in the tab is working
- `?` an agent is waiting for you (blocked)
- `✓` an agent finished and you haven't looked yet (done)
- nothing when idle / unknown (configurable)

Plugin id `dvoraj75.cowbell`, installed with
`herdr plugin install dvoraj75/herdr-cowbell`.

## What herdr gives us (and doesn't)

Terms: **workspace** (a "Space" in the sidebar) → **tabs** → **panes**
(terminals). An agent runs in a pane; herdr rolls pane state up to the tab.

| Fact | Source | Consequence |
|---|---|---|
| The tab bar draws only the tab label (plus ` Z` when zoomed). There are no tokens, styles or per-tab icons. | `src/client/shell/tabs.rs` (`tab_label`) | The icon **has to be part of the label**, so we rename tabs. No colour except what emoji carry themselves. |
| `tab.list` returns `agent_status` per tab, already rolled up: blocked > done (idle and unseen) > working > idle > unknown. | `src/app/creation.rs` `tab_info`, `src/app/api_helpers.rs` `tab_attention_priority` | We don't compute the rollup ourselves; one `tab.list` call gives the full picture. |
| `tab.rename {tab_id, label}` sets a custom name. There is **no way back** to the automatic name. | `src/workspace/tab.rs` `set_custom_name` | Don't touch tabs that still have automatic names (their label is just their `number`) unless the user opts in. |
| `events.subscribe` takes any lifecycle type without filters (`tab.*`, `pane.*`, `workspace.*`, `pane.agent_status_changed`). Newline-delimited JSON over the socket at `$HERDR_SOCKET_PATH`. | `herdr api schema --json`, docs/socket-api | One long-lived connection, no polling. |
| If the subscriber falls behind, herdr sends `events_lost` and closes the connection. | docs/socket-api "Event subscriptions" | On any disconnect: reconnect, then resync everything with `tab.list`. |
| Looking at a done tab sets `pane.seen = true`, maybe without a `pane.agent_status_changed` event (*verify*). | `src/app/actions.rs` `mark_active_tab_seen` | Also resync on `tab.focused`, `pane.focused` and `workspace.focused`, or `✓` never clears. |
| `[[startup]]` hooks are one-shot and unsupervised. They rerun on server start and live handoff, not on config reload or plugin link. | docs/plugins "Startup hooks" | The startup hook launches a **detached daemon** and exits. The daemon must be a singleton (pidfile or lock in `$HERDR_PLUGIN_STATE_DIR`). |
| Plugin commands: argv only, cwd = plugin root, env includes `HERDR_BIN_PATH`, `HERDR_SOCKET_PATH`, `HERDR_PLUGIN_STATE_DIR`, `HERDR_PLUGIN_CONFIG_DIR`. No timeout; max 32 at once. | docs/plugins, `src/app/api/plugins/runtime.rs` | Fine for a daemon plus short control actions. |
| herdr-radar (installed) subscribes to `tab.renamed`, so every rename wakes it. | `lib/subscribe.js` in herdr-radar | Rename **only when the icon actually changes**. A spinner would mean a rename per frame, so keep it off by default. |
| Other plugins also rename tabs (`herdr-auto-title`, `herdr-automatic-rename`, `herdr-tab-smart-rename`, `herdr-bar`). | marketplace, 2026-10-29 | Our model of the tab's real name must handle someone else renaming it (below). Document the conflict in the README. |

## Design

### Label model

```
label = icon + separator + base        (separator default " ")
label = base                           (when the state shows no icon)
```

- `base` is the tab's real name, stored per `tab_id` in
  `$HERDR_PLUGIN_STATE_DIR/state.json` together with the last label **we**
  wrote.
- `strip(label)` removes a leading icon from **any** preset (plus custom icons
  and spinner frames) followed by the separator. Running strip more than once is
  safe, so we never end up with `? ? grundza`.
- On `tab.renamed`: if the new label equals the label we last wrote, it's our
  own echo, so ignore it. Otherwise the user or another plugin renamed the tab:
  `base = strip(new label)`, then re-apply the icon.
- Skip tabs whose label equals their automatic `number`, unless the
  `include_auto_named` setting is on.
- Tab closed → drop its state entry.

### Icon presets

| preset | working | blocked | done | idle | unknown |
|---|---|---|---|---|---|
| `ask` (default?) | `◐` | `?` | `✓` | — | — |
| `symbols` (same as herdr's `status_indicators = "symbols"`) | `◐` | `×` | `✓` | — | — |
| `emoji` (the only preset with colour) | `⏳` | `❓` | `✅` | — | — |
| `nerd` (needs a Nerd Font) | `` | `` | `` | — | — |
| `custom` | from config | | | | |

The optional spinner applies only to `working`: frames `◐◓◑◒`, interval at
least 1000 ms, off by default.

### Config: `$HERDR_PLUGIN_CONFIG_DIR/config.json`

```json
{
  "preset": "ask",
  "icons": { "working": "◐", "blocked": "?", "done": "✓", "idle": "", "unknown": "" },
  "separator": " ",
  "include_auto_named": false,
  "spinner": { "enabled": false, "frames": ["◐", "◓", "◑", "◒"], "interval_ms": 1000 }
}
```

`icons` overrides single states of the chosen preset. If the file is missing,
use the defaults. If it's invalid, log the error and keep the last good config.

### Components

```
herdr-plugin.toml
bin/cowbell.js      daemon: subscribe → resync → rename; singleton; exits when the socket closes for good
bin/ctl.js          control: start | stop | on | off | refresh | cycle-preset | status
lib/socket.js       NDJSON client for $HERDR_SOCKET_PATH (request/response + subscription stream)
lib/label.js        strip / compose / skip rules  (pure, unit-tested)
lib/icons.js        presets + config merge         (pure, unit-tested)
lib/state.js        state.json read/write (atomic: write tmp file, then rename)
test/*.test.js      node:test, no dependencies
```

Runtime: **Node ≥ 18, no npm dependencies, no build step.** Proposed, not yet
confirmed; see open questions. Talk to herdr through the raw socket
(`net.connect(HERDR_SOCKET_PATH)`, which also accepts a Windows named pipe)
rather than spawning `herdr` for every rename.

### Daemon loop

```
start:
  take the lock (pidfile in STATE_DIR); exit if another live daemon holds it
  load config + state
connect:
  events.subscribe [pane.agent_status_changed, pane.created, pane.closed,
                    pane.moved, pane.exited, pane.focused, tab.created,
                    tab.closed, tab.renamed, tab.moved, tab.focused,
                    workspace.focused, workspace.closed]
  resync()
on event:   debounce ~100 ms, then resync()
            (tab.renamed is handled first, to update base)
resync():
  tabs = tab.list
  for each tab: want = compose(base(tab), icon(tab.agent_status))
                if tab.label != want: tab.rename(tab.id, want); remember want
  drop state for tabs that no longer exist
on disconnect / events_lost: back off 0.5 s → 30 s and retry; give up and exit
  when the server is gone (startup hook brings us back)
on "off" / SIGTERM: restore every tab to its base, delete pidfile, exit
```

### Manifest (draft)

```toml
id = "dvoraj75.cowbell"
name = "Cowbell"
version = "0.1.0"
min_herdr_version = "0.9.0"      # check which version added the APIs we use
description = "Agent status on your tabs: ◐ working, ? waiting for you, ✓ done."
platforms = ["linux", "macos"]   # windows later

[[startup]]
command = ["node", "bin/ctl.js", "start"]

[[events]]                       # watchdog: restart the daemon if it died (herdr-radar pattern)
on = "pane.agent_detected"
command = ["node", "bin/ctl.js", "start"]

[[actions]]
id = "toggle"
title = "Cowbell: on/off"
command = ["node", "bin/ctl.js", "toggle"]

[[actions]]
id = "off"
title = "Cowbell: off and remove icons"
command = ["node", "bin/ctl.js", "off"]

[[actions]]
id = "refresh"
title = "Cowbell: refresh now"
command = ["node", "bin/ctl.js", "refresh"]

[[actions]]
id = "cycle-preset"
title = "Cowbell: next icon preset"
command = ["node", "bin/ctl.js", "cycle-preset"]
```

Users bind keys in their `config.toml`:

```toml
[[keys.command]]
key = "prefix+b"
type = "plugin_action"
command = "dvoraj75.cowbell.toggle"
description = "cowbell on/off"
```

## Phases

1. **Scaffold**: manifest, `package.json` (`"type": "module"`, `node --test`),
   `.gitignore`, empty modules.
2. **Pure logic + tests**: `label.js` and `icons.js`. Cases: strip any
   preset's icon, strip repeated icons, user rename while an icon is shown,
   auto-named tab skipped, emoji (2-cell) icons, empty `base`.
3. **Socket client + daemon** against the sandbox server (see Testing): resync,
   rename, echo suppression, reconnect.
4. **Control actions**: start/stop/toggle/off/refresh, the singleton lock, and
   off restoring every tab name.
5. **Presets + config**: `cycle-preset`, `config.json`, custom icons.
6. **Edge cases** (*verify* each in the sandbox): done → seen clears `✓`;
   server restart (startup hook reruns, daemon replaced); live handoff; a pane
   moved across tabs; a tab closed while a rename is in flight; two daemons
   racing.
7. **Optional spinner** (≥ 1 s, off by default).
8. **Release**: README with screenshots of each preset, CHANGELOG, tag
   `v0.1.0`. The repo already has the `herdr-plugin` topic, so the marketplace
   lists it once `herdr-plugin.toml` is on the default branch.

## Testing: always on a sandbox server

The dev session usually runs **inside a herdr pane**. `HERDR_SOCKET_PATH` is
set there and points at the **live** server. Any `herdr …` call, even with
`XDG_CONFIG_HOME` overridden, then changes your real tabs and plugin registry.
Use an isolated server:

```sh
SB=<short dir>            # keep it short: the socket path must fit in 108 bytes
mkdir -p "$SB/c" "$SB/s" && cd "$SB"
clean() { env -u HERDR_SOCKET_PATH -u HERDR_BIN_PATH -u HERDR_PANE_ID -u HERDR_TAB_ID \
              -u HERDR_WORKSPACE_ID -u HERDR_ENV -u HERDR_STARTUP_CWD \
              XDG_CONFIG_HOME=c XDG_STATE_HOME=s "$@"; }
clean herdr server &                 # headless server, its own socket at c/herdr/herdr.sock
clean herdr plugin list              # must say "No plugins installed" before you link anything
```

- With relative `XDG_*` paths, plugin commands get relative socket and state
  paths too, and those resolve from the plugin root. So link a copy of the
  plugin that lives in `$SB` itself, or use a short absolute `$SB`.
- Make agent states without a real agent:
  `clean herdr pane report-agent <pane> --source custom:test --agent test --state blocked`
  (states: `idle | working | blocked | unknown`; checked on 0.9.3). `done`
  can't be reported: it is `idle` on a pane you haven't looked at. Get it by
  going working → idle on a tab that isn't focused.
- Check what happened: `clean herdr tab list`,
  `clean herdr plugin log list --plugin dvoraj75.cowbell`.
- Stop the server: `clean herdr server stop`.
- Link into the real herdr only when the user says so.

## Open questions

1. Default preset: `ask` (`◐ ? ✓`) or `emoji` for colour?
2. Should `done` get an icon, or only `working` and `blocked`?
3. Runtime: Node (proposed: a long-lived socket client is simplest there) or
   Bash + `jq`?
4. Windows support in v1, or later?
5. Should we also offer a tab-bar right-side summary
   (`ui.tab_bar_right` command entry, e.g. `? grundza 2 · ◐ grundza 1`) as a
   non-renaming mode? herdr-radar already uses that area.

## Risks

- **Stale icons**: names are saved in the session, so if the daemon dies
  uncleanly the icons stay. Mitigation: strip on startup, the `off` action,
  and running strip more than once is safe.
- **Fighting other renamers**: two plugins renaming the same tab can loop.
  Mitigation: echo suppression, and treat foreign renames as the new base
  (don't re-fight them). If renames ping-pong, back off.
- **Waking other plugins / CPU**: rename only on icon changes; spinner off by
  default.
- **herdr API is v1 and young**: pin `min_herdr_version`, keep socket calls in
  `lib/socket.js`, re-check the changelog on upgrades.

## References

- Plugins: https://herdr.dev/tui/docs/plugins/
- Socket API: https://herdr.dev/tui/docs/socket-api/ (and
  `herdr api schema --json` for the exact shapes in your version)
- CLI: https://herdr.dev/tui/docs/cli-reference/
- herdr source: https://github.com/herdrdev/herdr (`src/app/api/plugins/`)
- Examples: https://github.com/ogulcancelik/herdr-plugin-examples
- A daemon-style plugin to learn from: https://github.com/hhdebb/herdr-radar
  (`lib/subscribe.js`, `bin/agent-state.js`)
