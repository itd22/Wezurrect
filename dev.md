# Wezurrect API Status vs. resurrect.wezterm

Comparison based on direct diff of Lua source, not just README claims.

- Upstream: `MLFlexer/resurrect.wezterm` (archived)
- Fork: `YedPool/Wezurrect`

## Verdict

**API-compatible.** All public functions from upstream still exist with the same names and signatures. Configs written against upstream's documented API continue to work unmodified.

## Module structure

| Module | Upstream | Fork | Status |
|---|---|---|---|
| `state_manager.lua` | ✅ | ✅ | kept, extended |
| `workspace_state.lua` | ✅ | ✅ | kept, extended |
| `window_state.lua` | ✅ | ✅ | kept, fixed |
| `tab_state.lua` | ✅ | ✅ | kept |
| `fuzzy_loader.lua` | ✅ | ✅ | kept, modified |
| `encryption.lua` | ✅ | ✅ | kept, modified |
| `pane_tree.lua` | ✅ | ✅ | kept |
| `utils.lua` | ✅ | ✅ | kept |
| `file_io.lua` | ✅ | ✅ | kept |
| `instance_manager.lua` | ❌ | ✅ new | added |
| `process_handlers.lua` | ❌ | ✅ new | added |
| `spec/*` (tests) | ❌ | ✅ new | added |

## Public API surface

All confirmed unchanged (same name, same call signature):

- `pub.save_state(state)`
- `pub.load_state(name, type)`
- `pub.periodic_save(opts)`
- `pub.event_driven_save(opts)`
- `pub.delete_state(file_path)`
- `pub.workspace_state.get_workspace_state()`
- `pub.workspace_state.restore_workspace(state, opts)`
- `pub.window_state.get_window_state(mux_win)`
- `pub.tab_state.get_tab_state(mux_tab)`
- `pub.fuzzy_loader`, `pub.encryption`, `pub.pane_tree` — exposed the same way

New, purely additive:

- `pub.setup(config, opts)` — one-call convenience wrapper (opt-in)
- `pub.instance_manager` — per-window instance state, retention, startup prompt
- `pub.process_handlers` — Claude Code `SessionStart` hook integration
- `pub._test` — internals exposed only for unit tests

## Behavior fixes (API-compatible, logic changed)

| Area | Upstream behavior | Fork behavior |
|---|---|---|
| `event_driven_save` | recursively re-triggers `periodic_save` on every event | proper change-detection via structure signature; save paths decoupled |
| Save loop robustness | unguarded; one failure can wedge the loop | wrapped in `pcall`, logs and emits `resurrect.error`, always reschedules |
| `delete_state` | raw path concatenation, no validation | rejects `..` traversal, absolute paths, non-`.json` targets |
| `window_state.lua` restore | `active_tab:activate()` unguarded | nil-checked, avoids error when no active tab |
| Filename sanitization | escapes only the path separator | escapes a wide set of unsafe/control characters |
| State saves | overwrite in place | rotates to `.bak` + timestamped `.backups/` with retention (`backup_retention_count`) |

## Practical takeaway

Swapping the plugin source URL from `MLFlexer/resurrect.wezterm` to `YedPool/Wezurrect` should be a drop-in replacement for existing configs, while also picking up the bug fixes and new opt-in features (setup wrapper, instance manager, Claude Code hooks, backups) at no cost to compatibility.



# Dependency, What is `chrisgve/dev.wezterm`?

A small **developer utility plugin** for WezTerm plugin authors. It is not a
session/state manager itself — it's infrastructure that other plugins (like
Wezurrect) use internally.

It was pulled in as a new dependency by `YedPool/Wezurrect`'s `init.lua`
(`wezterm.plugin.require("https://github.com/chrisgve/dev.wezterm")`) and is
not present in upstream `MLFlexer/resurrect.wezterm`.

## The problem it solves

WezTerm installs plugins into a hashed, cache-mangled directory name — something
like `...sZsYedPoolsZsWezurrect`. There's no built-in way for a plugin to
reliably find *its own* install path at runtime. `dev.wezterm` exists to solve
exactly that one problem.

## What it actually does

- `wezterm.plugin.list()` — enumerates all installed plugins
- Searches that list for one whose directory name contains a set of
  **keywords** the caller provides (Wezurrect passes `{"YedPool"}`)
- Once found, returns the plugin's real filesystem path and wires up
  `package.path` so `require("resurrect.xxx")` resolves correctly
- Nothing else — no file I/O beyond reading Lua's own plugin registry, no
  shell exec, no network calls

## Why Wezurrect needs it

Upstream `resurrect.wezterm` derived its state directory in a simpler way and
never needed this. Wezurrect uses `dev.setup()` to reliably locate its own
plugin folder so it can correctly build the path to its `state/` directory,
regardless of how WezTerm renamed the install folder on disk.

A comment in Wezurrect's `init.lua` explains the specific reason: depending on
which URL you install from, WezTerm encodes the plugin path differently —
either `...sZsYedPoolsZsWezurrect` (canonical URL) or
`...sZsYedPoolsZsresurrectsDswezterm` (the README's redirected URL). `"YedPool"`
is the one substring common to both forms, which is why that's the keyword
used.

## Security scan result

~300 lines total. Reviewed the full source:

- Only does string matching over an in-memory plugin list and sets
  `package.path`
- No network calls
- No shell/process execution
- No file writes beyond what Lua's `require` machinery already does

**Verdict:** legitimate, narrowly-scoped helper plugin. The one thing worth
noting isn't a security risk — it's a supply-chain consideration: installing
Wezurrect now means trusting a second author's code (`chrisgve`) in addition
to `YedPool`, since `dev.wezterm` loads and runs alongside Wezurrect itself.
