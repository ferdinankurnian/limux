---
name: limux-a2a
description: Use inside Limux when a pane needs to identify its own workspace, pane, tab, or surface IDs; create new terminal panes or workspaces; launch commands into those targets; send structured messages between panes or workspaces; submit text with keys; read peer terminal output; or notify the human through the Limux CLI.
---

# Limux A2A

Use this skill from inside Limux to coordinate live terminal panes and workspaces through the `limux` CLI.

Do not depend on generated files or persistent rosters. Treat Limux itself as the source of truth: identify the current pane from `LIMUX_*`, discover peers through the control socket, and address exact surfaces when sending messages.

Assume the installed `limux` command is on `PATH`. Use local `./target/.../limux-cli` only when explicitly testing this repo build.

## Identity

Every Limux-spawned terminal should know its own live IDs through environment variables:

```bash
printf 'workspace_id=%s\nsurface_id=%s\npane_id=%s\ntab_id=%s\nsocket=%s\n' \
  "$LIMUX_WORKSPACE_ID" \
  "$LIMUX_SURFACE_ID" \
  "$LIMUX_PANE_ID" \
  "$LIMUX_TAB_ID" \
  "$LIMUX_SOCKET"
```

Meanings:

- `LIMUX_WORKSPACE_ID`: current workspace UUID.
- `LIMUX_SURFACE_ID`: exact terminal surface ID, usually `<pane_id>:<tab_id>`.
- `LIMUX_PANE_ID`: current pane ID.
- `LIMUX_TAB_ID`: current tab ID.
- `LIMUX_SOCKET`: control socket path used by the CLI.

Fallback if environment variables are missing or suspicious:

```bash
limux --json identify
limux --json list-workspaces
```

If `identify` cannot resolve caller fields, use `list-panels` for the workspace and match by focused surface only as a fallback:

```bash
limux --json list-panels --workspace "$LIMUX_WORKSPACE_ID"
```

## Discover Live Targets

List all workspaces:

```bash
limux --json list-workspaces
```

List panes and surfaces in the current workspace:

```bash
limux --json list-panes --workspace "$LIMUX_WORKSPACE_ID"
limux --json list-panels --workspace "$LIMUX_WORKSPACE_ID"
```

List surfaces in another workspace:

```bash
limux --json list-panels --workspace "<workspace-name-or-id>"
```

Routing rule:

- Use `--surface <surface-id>` for exact terminal-to-terminal messages.
- Add `--workspace <workspace-id-or-name>` when the target surface is outside the current workspace.
- Avoid workspace-only sends unless there is exactly one obvious active terminal in that workspace.

## Create A New Pane

Create a terminal pane next to the current pane and launch a command:

```bash
limux --json new-pane --direction right --command "bash"
```

Directions: `left`, `right`, `up`, `down`.

When focus may have moved, pass the source explicitly:

```bash
limux --json new-pane \
  --workspace "$LIMUX_WORKSPACE_ID" \
  --surface "$LIMUX_SURFACE_ID" \
  --pane "$LIMUX_PANE_ID" \
  --direction right \
  --command "bash"
```

The JSON response includes the new `workspace_id`, `pane_id`, and `surface_id`. Capture `surface_id` immediately if you plan to send text there.

Example:

```bash
created="$(limux --json new-pane --direction right --command "bash")"
target_surface="$(printf '%s\n' "$created" | jq -r '.surface_id')"
printf 'new_surface=%s\n' "$target_surface"
```

Limits:

- Live GTK `new-pane` supports terminal panes.
- Browser pane creation is not live-bridge supported yet.
- `--command` is injected into the new terminal and submitted.

## Launch A Pane With A Task

For CLI tools that accept an initial prompt as an argument, include the task prompt directly in `--command`. This avoids a separate handshake just to start work.

Pattern:

```bash
parent_surface="$LIMUX_SURFACE_ID"
created="$(limux --json new-pane --direction right --command 'codex "You are working in a Limux child pane. Parent surface: '"$parent_surface"'. When you have status or a final result, reply with: limux send --surface '"$parent_surface"' <message>. Task: inspect the current diff and report blocking issues only."')"
child_surface="$(printf '%s\n' "$created" | jq -r '.surface_id')"
```

Use the same idea for interactive and non-interactive tools:

```bash
limux --json new-pane --direction right --command 'codex "Task prompt here. Reply to parent with limux send --surface <parent-surface> ..."'
limux --json new-pane --direction right --command 'codex exec "Task prompt here. Reply to parent with limux send --surface <parent-surface> ..."'
limux --json new-pane --direction right --command 'claude "Task prompt here. Reply to parent with limux send --surface <parent-surface> ..."'
limux --json new-pane --direction right --command 'claude -p "Task prompt here. Reply to parent with limux send --surface <parent-surface> ..."'
```

Sandbox note:

- The launched process must be able to connect to the Limux socket to call back with `limux send`.
- Codex tool execution may run inside a sandbox that cannot connect to `/tmp/...` or `/run/user/...` Limux sockets by default.
- If a Codex child reports `failed to connect to socket`, the spawn worked but the child does not have socket access. Use an approved socket-access profile or have the parent inject follow-up text instead of expecting the child tool to call `limux send`.

Keep the launched prompt self-contained:

- Include the literal parent surface ID.
- Include the exact reply command.
- Include the task and expected output.
- Tell the child to keep replies short and send file paths for large context.

The parent does not need a readiness handshake to learn child IDs; `new-pane` already returns the child `workspace_id`, `pane_id`, and `surface_id`.

## Create A New Workspace

Create an isolated workspace and launch a command in its first terminal:

```bash
limux --json new-workspace --cwd "$PWD" --command "bash"
```

Capture the returned workspace ID:

```bash
created="$(limux --json new-workspace --cwd "$PWD" --command "bash")"
target_workspace="$(printf '%s\n' "$created" | jq -r '.workspace_id')"
printf 'new_workspace=%s\n' "$target_workspace"
```

Rename it if a human-readable target name helps:

```bash
limux rename-workspace --workspace "$target_workspace" "worker-shell"
```

Discover its terminal surface before sending text:

```bash
limux --json list-panels --workspace "$target_workspace"
```

Use the same task-launch pattern for workspace isolation:

```bash
parent_surface="$LIMUX_SURFACE_ID"
limux --json new-workspace \
  --cwd "$PWD" \
  --command 'codex "Work in this isolated Limux workspace. Parent surface: '"$parent_surface"'. Reply with: limux send --surface '"$parent_surface"' <message>. Task: run the test suite and summarize failures."'
```

## Send Text

Send to an exact surface in the current workspace:

```bash
limux send --surface "<target-surface-id>" "hello"
```

Send to an exact surface in another workspace:

```bash
limux send --workspace "<target-workspace>" --surface "<target-surface-id>" "hello"
```

Send a submitted multi-line request by including a trailing newline:

```bash
limux send --surface "<target-surface-id>" $'<limux-msg from-surface="'"$LIMUX_SURFACE_ID"'" to-surface="<target-surface-id>" id="'"$(uuidgen)"'" ts="'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'">\n<request>Run pwd and report the result.</request>\n</limux-msg>\n'
```

Use a structured envelope for machine-readable coordination:

```xml
<limux-msg from-surface="<sender-surface-id>" to-surface="<target-surface-id>" id="<uuid>" ts="<iso8601>" reply-to="<optional-parent-id>">
  <context>Short reason for the request.</context>
  <request>The exact task or message.</request>
  <expect>What kind of response is needed.</expect>
</limux-msg>
```

Envelope rules:

- Use surface IDs, not names, for `from-surface` and `to-surface`.
- Generate `id` with `uuidgen`.
- Generate `ts` with `date -u +%Y-%m-%dT%H:%M:%SZ`.
- Replies set `reply-to` to the original `id`.
- Keep messages short; write long context to a shared file and send the path.

## Parallel Children

A parent can launch many panes or workspaces in parallel. Capture each returned `surface_id`.

Example:

```bash
parent_surface="$LIMUX_SURFACE_ID"
review_json="$(limux --json new-pane --direction right --command 'codex "Review the current diff. Parent surface: '"$parent_surface"'."')"
test_json="$(limux --json new-pane --direction down --command 'codex exec "Run focused tests. Parent surface: '"$parent_surface"'."')"

review_surface="$(printf '%s\n' "$review_json" | jq -r '.surface_id')"
test_surface="$(printf '%s\n' "$test_json" | jq -r '.surface_id')"
```

If children are doing related work, make them aware of sibling surfaces so they can coordinate directly:

```bash
limux send --surface "$review_surface" "Sibling test surface: $test_surface. Coordinate directly if your findings need test confirmation."
limux send --surface "$test_surface" "Sibling review surface: $review_surface. Coordinate directly if test failures need code inspection."
```

When sibling IDs are not known until after all spawns complete, start children with parent-only prompts, then send a sibling map to each child.

## Submit Keys

`limux send` injects text. If the target terminal needs the current line submitted, send Enter:

```bash
limux send-key --surface "<target-surface-id>" enter
```

For another workspace:

```bash
limux send-key --workspace "<target-workspace>" --surface "<target-surface-id>" enter
```

Common keys:

```bash
limux send-key --surface "<target-surface-id>" enter
limux send-key --surface "<target-surface-id>" tab
limux send-key --surface "<target-surface-id>" escape
limux send-key --surface "<target-surface-id>" '<Ctrl>c'
limux send-key --surface "<target-surface-id>" '<Ctrl>d'
```

## Read Output

Inspect a target terminal:

```bash
limux read-screen --surface "<target-surface-id>" --lines 80
limux capture-pane --surface "<target-surface-id>" --lines 80
```

For another workspace:

```bash
limux read-screen --workspace "<target-workspace>" --surface "<target-surface-id>" --lines 80
```

Check whether a surface is alive before assuming it is stuck:

```bash
limux --json surface-health --workspace "<target-workspace>" --surface "<target-surface-id>"
```

## Notify The Human

Use `notify` when human attention is needed:

```bash
limux notify --workspace "$LIMUX_WORKSPACE_ID" \
  --subtitle "input needed" \
  --body "A pane is blocked and needs a decision" \
  "Limux task needs attention"
```

## Practical Flows

Create a scratch pane, send it a command, and read the result:

```bash
created="$(limux --json new-pane --direction right --command "bash")"
surface="$(printf '%s\n' "$created" | jq -r '.surface_id')"
limux send --surface "$surface" $'pwd\n'
limux read-screen --surface "$surface" --lines 20
```

Create an isolated workspace, address its first surface, and send a request:

```bash
created="$(limux --json new-workspace --cwd "$PWD" --command "bash")"
workspace="$(printf '%s\n' "$created" | jq -r '.workspace_id')"
surface="$(limux --json list-panels --workspace "$workspace" | jq -r '.surfaces[0].surface_id')"
limux send --workspace "$workspace" --surface "$surface" $'echo ready\n'
```

Reply to a structured message:

```bash
limux send --surface "<original-sender-surface-id>" $'<limux-msg from-surface="'"$LIMUX_SURFACE_ID"'" to-surface="<original-sender-surface-id>" id="'"$(uuidgen)"'" reply-to="<original-id>" ts="'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'">\n<response>Done.</response>\n</limux-msg>\n'
```

## Failure Handling

- `failed to connect to socket`: check `printf '%s\n' "$LIMUX_SOCKET"` and run `limux list-workspaces`; the host may not be running.
- `workspace not found`: run `limux --json list-workspaces`; use the exact UUID or name.
- `terminal surface not found`: run `limux --json list-panels --workspace ...`; surfaces change when panes/tabs are recreated.
- Text appears but does not run: include a trailing newline or send `limux send-key ... enter`.
- Target is silent: run `surface-health`, then `read-screen`, then resend with `reply-to` if needed.
