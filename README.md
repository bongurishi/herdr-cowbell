# herdr-cowbell

[![CI](https://github.com/dvoraj75/herdr-cowbell/actions/workflows/ci.yml/badge.svg)](https://github.com/dvoraj75/herdr-cowbell/actions/workflows/ci.yml)
[![Latest Release](https://img.shields.io/github/v/release/dvoraj75/herdr-cowbell)](https://github.com/dvoraj75/herdr-cowbell/releases)
[![License](https://img.shields.io/github/license/dvoraj75/herdr-cowbell)](LICENSE)
![Minimum herdr version](https://img.shields.io/badge/herdr-%3E%3D0.9.2-blue)
![Node.js](https://img.shields.io/badge/node-%3E%3D18-brightgreen)
[![Documentation](https://img.shields.io/badge/docs-documentation-blue)](DOCS_URL)

<!-- Replace with the approved hero GIF from issue #20 -->
![herdr-cowbell demo](MEDIA_HERO_GIF_URL)

**Know what your agents are doing, without leaving your terminal.**

herdr-cowbell is a herdr plugin that displays agent status directly in your terminal's tab names. Instantly recognize working, blocked, and completed agents through customizable icons, five visual presets, and keyboard-driven actions. Lightweight, dependency-free, and designed to fit naturally into your herdr workflow.

## Features

- **Live agent status:** Identify working, blocked, and completed agents at a glance.
- **Five visual presets:** Choose `ask`, `symbols`, `emoji`, `nerd`, or `custom`.
- **Keyboard actions:** Toggle Cowbell, refresh status, and cycle presets through herdr actions.
- **Configurable:** Customize icons, separators, and automatic tab-name handling.

## Install

Requirements: herdr 0.9.2 or newer, Node.js 18 or newer on `PATH` as `node`, and Linux or macOS.

```sh
herdr plugin install dvoraj75/herdr-cowbell
```

herdr runs no hook on install, link, or enable. The daemon starts at the next herdr server start, when the next agent is detected in a pane, or immediately with **Cowbell: refresh now**.

By default, Cowbell leaves tabs you never named (auto-named tabs) alone.

## Quickstart

1. Install Cowbell using the command above.
2. Open a herdr session with agent panes.
3. Run **Cowbell: refresh now** to start the daemon immediately.
4. Press your configured key binding to toggle Cowbell.

The default preset is `ask`:

| Agent status | Icon | Meaning |
|---|---|---|
| Working | `◐` | Agent is working |
| Blocked | `?` | Waiting for you |
| Done | `✓` | Completed and not yet seen |

Idle and unknown agents show no icon.

### Key binding

Add this example to your herdr `config.toml`:

```toml
[[keys.command]]
key = "prefix+shift+b"
type = "plugin_action"
command = "dvoraj75.cowbell.toggle"
description = "cowbell on/off"
```

Apply the binding without restarting the server:

```sh
herdr server reload-config
```

Avoid `prefix+b`: in herdr 0.9.3 it is the default sidebar toggle, and a plugin binding on it silently replaces that toggle.

## Presets

| Preset | Working | Blocked | Done |
|---|---|---|---|
| `ask` (default) | `◐` | `?` | `✓` |
| `symbols` | `◐` | `×` | `✓` |
| `emoji` | `⏳` | `❓` | `✅` |
| `nerd` | `` | `` | `` |
| `custom` | Configurable | Configurable | Configurable |

**Nerd Font required:** The `nerd` preset requires a compatible Nerd Font.

### Preset captures

The following captures show the actual tab labels and condensed `tab.list` replies from a sandbox herdr 0.9.3 server.

### ask

```text
◐ api     ? review     ✓ tests     notes     shell    +
```

[Capture details](docs/captures/ask.txt)

<!-- TODO: Add screenshot from Demo media, issue #20 -->

### symbols

```text
◐ api     × review     ✓ tests     notes     shell    +
```

[Capture details](docs/captures/symbols.txt)

<!-- TODO: Add screenshot from Demo media, issue #20 -->

### emoji

```text
⏳ api     ❓ review     ✅ tests     notes     shell    +
```

[Capture details](docs/captures/emoji.txt)

<!-- TODO: Add screenshot from Demo media, issue #20 -->

### nerd

```text
 api      review      tests     notes     shell    +
```

[Capture details](docs/captures/nerd.txt)

<!-- TODO: Add screenshot from Demo media, issue #20 -->

### custom

Example configuration:

```json
{
  "preset": "custom",
  "icons": {
    "working": "»",
    "blocked": "!",
    "done": "*"
  }
}
```

```text
» api     ! review     * tests     notes     shell    +
```

[Capture details](docs/captures/custom.txt)

<!-- TODO: Add screenshot from Demo media, issue #20 -->

**Cowbell: next icon preset** cycles:

`ask → symbols → emoji → nerd → custom → ask`

The custom preset is skipped unless `icons` defines at least one non-empty icon.

## Actions

All action IDs use the prefix `dvoraj75.cowbell.`.

| Action ID | Title | Description |
|---|---|---|
| `toggle` | Cowbell: on/off | Toggles Cowbell on or off. |
| `off` | Cowbell: off and remove icons | Stops the daemon, restores tab names, and stays off across restarts. |
| `refresh` | Cowbell: refresh now | Resynchronizes status and starts the daemon if needed. Does nothing while Cowbell is off. |
| `cycle-preset` | Cowbell: next icon preset | Switches to the next preset and applies it immediately. |

## Configuration

Configuration is stored in `config.json` inside the plugin configuration directory.

Normally:

```text
~/.config/herdr/plugins/config/dvoraj75.cowbell/config.json
```

Use `herdr plugin list` to find the plugin configuration directory.

| Key | Type | Default | Description |
|---|---|---|---|
| `preset` | string | `"ask"` | Visual preset. |
| `icons` | object | `{}` | Per-status icon overrides. |
| `separator` | string | `" "` | Text between the icon and tab name. |
| `include_auto_named` | boolean | `false` | Also adds icons to automatically named tabs. |
| `spinner.enabled` | boolean | `false` | Reserved; no effect in 0.1.0. |
| `spinner.frames` | array | `["◐","◓","◑","◒"]` | Reserved; no effect in 0.1.0. |
| `spinner.interval_ms` | number | `1000` | Reserved; no effect in 0.1.0. |

Icon overrides support `working`, `blocked`, `done`, `idle`, and `unknown`. An empty string removes the icon for that state.

**Configuration changes:** The next icon preset action reloads the configuration. A manual edit requires a daemon restart or switching Cowbell off and on.

The spinner options are validated but unused in version 0.1.0. They are reserved for possible future functionality.

## Conflicts with other tab renamers

Other plugins may also rename tabs, including:

- herdr-auto-title
- herdr-automatic-rename
- herdr-tab-smart-rename
- herdr-bar

Cowbell treats a rename by another plugin or by you as the tab's new name and adds its icon in front. It removes a leading icon from any preset first to prevent duplicate icons.

If another plugin repeatedly overwrites Cowbell's rename, Cowbell temporarily pauses updates for that tab and retries with increasing backoff.

Plugins listening to `tab.renamed` may also wake on Cowbell's icon changes. Ideally, only one plugin should rename a given tab.

## Known limitations

- **Other tab renamers:** Repeated rename conflicts can temporarily hide Cowbell icons.
- **Disabling or uninstalling:** herdr does not run plugin hooks on disable or uninstall. Run **Cowbell: off and remove icons** first.
- **Initial startup:** Cowbell can be enabled before its daemon starts. Use **Cowbell: refresh now** to start it.
- **Auto-named tabs:** herdr cannot restore an automatic tab name after it has been renamed. `include_auto_named` is therefore opt-in.
- **Server restart:** The startup hook replaces the running daemon, restoring icons shortly afterward.
- **Rare lock races:** Concurrent daemon starts or stale lock files can occasionally lose a preset step. Normal operation recovers automatically.
- **Refresh while off:** Refresh does nothing while Cowbell is switched off.
- **Manual configuration edits:** Require a daemon restart.
- **Spinner:** Not implemented in version 0.1.0.

## Uninstalling

herdr does not run plugin hooks when disabling or uninstalling a plugin.

Before removing Cowbell, run:

**Cowbell: off and remove icons**

This stops the daemon and restores the tab names.

Then disable or uninstall it using herdr's plugin commands.

If the daemon is not running when the plugin is removed, existing icons may remain in tab names.

## Documentation

For the complete documentation, installation guidance, configuration reference, troubleshooting, and examples, visit:

**[herdr-cowbell Documentation](DOCS_URL)**

## Development

Cowbell has no build step and no external dependencies.

Run the test suite:

```sh
node --test
```

**Important:** Never run herdr commands against your live session while developing. Use an isolated sandbox server.

See [CONTRIBUTING.md](.github/CONTRIBUTING.md) for development setup, testing, and contribution guidelines.

## Project structure

| File | Purpose |
|---|---|
| `bin/cowbell.js` | Daemon entry point. |
| `bin/ctl.js` | Control CLI for actions and hooks. |
| `lib/churn.js` | Rename-conflict bookkeeping. |
| `lib/config.js` | Configuration reading, writing, and preset cycling. |
| `lib/control.js` | Control commands. |
| `lib/daemon.js` | Event loop, subscriptions, synchronization, and reconnection. |
| `lib/icons.js` | Presets, defaults, and configuration validation. |
| `lib/label.js` | Tab-label logic. |
| `lib/lock.js` | Singleton process lock. |
| `lib/socket.js` | herdr socket client. |
| `lib/state.js` | State file and directory management. |

## Community

- [Report a bug](https://github.com/dvoraj75/herdr-cowbell/issues/new)
- [Request a feature](https://github.com/dvoraj75/herdr-cowbell/issues/new)
- [Ask a question](https://github.com/dvoraj75/herdr-cowbell/discussions)
- [Contribute](.github/CONTRIBUTING.md)
- [Security policy](SECURITY.md)
- [Code of Conduct](CODE_OF_CONDUCT.md)

## License

MIT. See [LICENSE](LICENSE).
