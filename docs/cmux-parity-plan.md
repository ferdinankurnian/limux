# cmux-parity plan

Bringing the cmux CLI feature set (and agent-to-agent comms workflow) to limux.

## What limux already has

| Feature | Status | Location |
|---|---|---|
| Control socket + JSON envelope | ✅ | `limux-control`, `limux-protocol` |
| `limux identify` | ✅ | `limux-cli/src/main.rs` |
| `limux send` (send keys to a surface) | ✅ | ditto |
| `limux read-screen` / `capture-pane` | ✅ | ditto |
| `limux list-panes` / `list-workspaces` / `list-panels` | ✅ | ditto |
| `limux new-workspace` / `close-workspace` / `new-surface` / `new-pane` | ✅ | ditto |
| `limux rename-*` / `tab-action` | ✅ | ditto |
| `limux browser …` (click/type/snapshot/screenshot) | ✅ | ditto |
| OSC 9 / desktop notification callback | ✅ | `terminal.rs` + `pane.rs` |
| Bell + workspace unread highlight | ✅ | `window.rs` |

## Gaps vs. cmux

1. **Env auto-wiring** — spawned shells don't see `LIMUX_WORKSPACE` / `LIMUX_SURFACE` / `LIMUX_PANE` / `LIMUX_SOCKET`, so every CLI call has to be told which target. cmux's magic is that you just type `cmux notify …` and it targets the current pane.
2. **`limux notify`** — native notification with title/body/subtitle, routed into sidebar and toast.
3. **`limux progress`** — 0.0–1.0 bar with label, per-surface.
4. **`limux log`** — structured per-workspace log stream with level + source.
5. **`limux markdown`** — open `.md` in a live-reload GTK panel.
6. **`limux claude-hook`** — wrapper that wires Claude Code session-start / stop / notification events into the UI (plus OpenCode/Gemini variants).
7. **Agent-team spawner** — spawn N agent surfaces (codex, claude-code, opencode, gemini) into a workspace with a pre-seeded `AGENTS.md` describing the XML message protocol so they can talk to each other.
8. **Per-pane attention ring + sidebar metadata** (git branch, PR, cwd, ports, latest notif text) — nice-to-have, not the user's priority.

## Phased delivery

### Phase 1 — Env auto-wiring (foundation)  ← START HERE
- Thread `workspace_id` / `surface_id` / `pane_id` into `TerminalOptions`.
- Build `ghostty_env_var_s` array at `create_terminal`, set `env_vars` + `env_var_count` on `ghostty_surface_config_s`.
- Exported vars: `LIMUX_WORKSPACE`, `LIMUX_SURFACE`, `LIMUX_PANE`, `LIMUX_SOCKET`.
- CLI resolver: if `--workspace` / `--surface` not given, fall back to env vars, then to `identify`.
- Tests: spawn a pane, `env | grep LIMUX_` sees all four.

### Phase 2 — `limux notify` + `limux progress` + `limux log`
- New control commands: `notification.create`, `progress.set`, `log.append`.
- `notify` plumbs through existing unread-workspace pipeline + libadwaita toast.
- `progress` renders a small bar in the sidebar row for the surface.
- `log` writes to `~/.local/share/limux/logs/<workspace>.log` and emits a tail to the sidebar on demand.

### Phase 3 — `limux markdown <file>`
- `markdown.open` control command spawns a WebKit pane (we already have browser panes) pointing at a local render URL, with an inotify watcher that reloads on change.

### Phase 4 — `limux claude-hook`
- Subcommand that reads Claude Code hook JSON from stdin and fans out to `notify` / `log` / `progress`.
- Writes install snippet to `~/.claude/settings.json` hooks section.
- Parallel helpers: `limux opencode-hook`, `limux gemini-hook`.

### Phase 5 — `limux agent-team`
- `limux agent-team --workspace <id> --agents codex,claude-code[,opencode,gemini]`
- For each agent: create a surface, spawn the agent CLI in it, record its surface id.
- Seed `./AGENTS.md` in the workspace cwd with the XML message protocol and the surface-id table so agents know who they're talking to.
- The prompt the user referenced ("run `limux -h`, identify yourself and your peers, codify in AGENTS.md") becomes the default seed.

## Non-goals (for this branch)
- SSH workspaces — separate branch.
- Browser import (cookies from Chrome/Firefox/Arc) — separate branch.
- Per-pane blue attention ring in GTK — nice-to-have follow-up after phase 1–2 land.
