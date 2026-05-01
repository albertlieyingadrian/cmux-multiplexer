---
name: cmux
description: cmux terminal multiplexer - manage workspaces, spawn agents, restore sessions. USE WHEN user says "restore sessions", "cmux workspaces", "spawn workspace", "list workspaces", "cmux status".
---

# cmux

Manage cmux workspaces and Claude Code agents from one terminal.

## Quick Reference

```bash
# List workspaces
cmux list-workspaces

# Create named workspace with Claude + prompt (no focus steal)
CURRENT=$(cmux current-workspace 2>&1 | awk '{print $1}')
NEW_UUID=$(cmux new-workspace --command "claude 'Your prompt here'" 2>&1 | awk '{print $2}')
cmux rename-workspace --workspace "$NEW_UUID" "workspace-name" 2>&1
cmux select-workspace --workspace "$CURRENT" 2>&1

# Resume session in named workspace (no focus steal)
CURRENT=$(cmux current-workspace 2>&1 | awk '{print $1}')
NEW_UUID=$(cmux new-workspace --command "claude --resume <session-id>" 2>&1 | awk '{print $2}')
cmux rename-workspace --workspace "$NEW_UUID" "workspace-name" 2>&1
cmux select-workspace --workspace "$CURRENT" 2>&1

# Rename / Select / Close
cmux rename-workspace --workspace workspace:N "Name"
cmux select-workspace --workspace workspace:N
cmux close-workspace --workspace workspace:N
```

## Key Patterns

- **Send command to any workspace:** `cmux send --workspace workspace:N "text" && cmux send-key --workspace workspace:N Enter`
- **Read any agent's screen:** `cmux read-screen --workspace workspace:N`
- **Spawn named workspace:** use `spawn-workspace.sh` script below

## Surface (Tab) Management

```bash
# List panes and surfaces
cmux list-panes --workspace workspace:N
cmux list-pane-surfaces --workspace workspace:N

# Move / Reorder / Rename / Close
cmux move-surface --surface surface:S --workspace workspace:N
cmux reorder-surface --surface surface:S --index 0
cmux rename-tab --surface surface:S "New Name"
cmux close-surface --surface surface:S --workspace workspace:N

# Split pane
cmux drag-surface-to-split --surface surface:S left|right|up|down
cmux break-pane --workspace workspace:N --pane pane:M
```

## Scripts

### spawn-workspace.sh

Save to `.claude/skills/cmux/scripts/spawn-workspace.sh` and `chmod +x`.

```bash
#!/bin/bash
# Spawn a named cmux workspace with Claude Code (no focus steal)
# Usage: spawn-workspace.sh <name> [--prompt "..."] [--loop [pct]]
#
# --loop       Enable infinite loop (auto-handoff at 60% context)
# --loop 70    Enable infinite loop at 70% context

set -e

NAME="${1:?Usage: spawn-workspace.sh <name> [--prompt \"...\"] [--loop [pct]]}"
shift

PROMPT=""
LOOP_PCT=""

while [[ $# -gt 0 ]]; do
  case "$1" in
    --prompt) PROMPT="$2"; shift 2 ;;
    --loop)
      if [[ -n "$2" ]] && [[ "$2" =~ ^[0-9]+$ ]]; then
        LOOP_PCT="$2"; shift 2
      else
        LOOP_PCT="60"; shift
      fi
      ;;
    *) echo "Unknown arg: $1"; exit 1 ;;
  esac
done

CURRENT=$(cmux current-workspace 2>&1 | awk '{print $1}')

if [[ -n "$PROMPT" ]]; then
  TMPFILE=$(mktemp /tmp/spawn-prompt.XXXXXX)
  echo "$PROMPT" > "$TMPFILE"
  NEW_UUID=$(cmux new-workspace --command "claude \"\$(cat $TMPFILE)\"" 2>&1 | awk '{print $2}')
else
  NEW_UUID=$(cmux new-workspace --command "claude" 2>&1 | awk '{print $2}')
fi

cmux rename-workspace --workspace "$NEW_UUID" "$NAME" 2>&1

# Register loop config for this workspace
if [[ -n "$LOOP_PCT" ]]; then
  LOOPS_DIR="$HOME/.claude/loops"
  mkdir -p "$LOOPS_DIR"
  echo "{\"threshold\": $LOOP_PCT}" > "$LOOPS_DIR/ws-${NEW_UUID}.json"
  echo "Loop registered at ${LOOP_PCT}% for workspace $NAME"
fi

cmux select-workspace --workspace "$CURRENT" 2>&1
echo "Workspace '$NAME' ready"
```

### cmux-session-map.py (Hook)

Save to `.claude/hooks/cmux-session-map.py` and `chmod +x`. Register as a hook for `SessionStart` and `SessionEnd`.

Maps Claude Code sessions to cmux surfaces so you can track which session is in which workspace.

```python
#!/usr/bin/env python3
"""Map Claude Code sessions to cmux surfaces/workspaces.
Maintains /tmp/cmux-session-map.json with active mappings."""

import json
import os
import sys
import fcntl
from datetime import datetime

MAP_FILE = "/tmp/cmux-session-map.json"

def load_map():
    try:
        with open(MAP_FILE) as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return {}

def save_map(data):
    with open(MAP_FILE, "w") as f:
        fcntl.flock(f, fcntl.LOCK_EX)
        json.dump(data, f, indent=2)
        f.write("\n")
        fcntl.flock(f, fcntl.LOCK_UN)

def handle_start(payload):
    session_id = payload.get("session_id")
    surface_id = os.environ.get("CMUX_SURFACE_ID")
    workspace_id = os.environ.get("CMUX_WORKSPACE_ID")
    if not session_id or not surface_id:
        return
    data = load_map()
    data[surface_id] = {
        "session_id": session_id,
        "workspace_id": workspace_id or "",
        "cwd": payload.get("cwd", ""),
        "started": datetime.now().isoformat(),
    }
    save_map(data)

def handle_end(payload):
    surface_id = os.environ.get("CMUX_SURFACE_ID")
    if not surface_id:
        return
    data = load_map()
    if surface_id in data:
        del data[surface_id]
        save_map(data)

if __name__ == "__main__":
    try:
        payload = json.load(sys.stdin)
        event = payload.get("hook_event_name")
        if event == "SessionStart":
            handle_start(payload)
        elif event == "SessionEnd":
            handle_end(payload)
    except Exception:
        pass
```

## Browser Automation

The `cmux browser` command group provides browser automation against cmux browser surfaces. Use it to navigate, interact with DOM elements, inspect page state, evaluate JavaScript, and manage browser session data.

Full docs: https://www.cmux.dev/docs/browser-automation

### Targeting Surfaces

Most subcommands require a target surface. Pass it positionally or with `--surface`:

```bash
cmux browser open https://example.com                          # Opens in caller's workspace
cmux browser surface:2 url                                      # Get URL of surface:2
cmux browser --surface surface:2 navigate https://example.org   # Navigate specific surface
```

### Navigation

```bash
cmux browser open [url]                    # Create browser split (or navigate if surface given)
cmux browser open-split [url]              # Open in new split
cmux browser goto|navigate <url> [--snapshot-after]
cmux browser back|forward|reload [--snapshot-after]
cmux browser url|get-url                   # Get current URL
```

### Waiting

```bash
cmux browser wait --selector <css>         # Wait for element
cmux browser wait --text <text>            # Wait for text to appear
cmux browser wait --url-contains <text>    # Wait for URL fragment
cmux browser wait --load-state <interactive|complete>
cmux browser wait --function <js>          # Wait for JS condition
cmux browser wait --timeout-ms <ms>        # Custom timeout
```

### DOM Interaction

All mutating actions support `--snapshot-after` for verification.

```bash
cmux browser click <selector> [--snapshot-after]
cmux browser dblclick <selector>
cmux browser hover <selector>
cmux browser focus <selector>
cmux browser check|uncheck <selector>
cmux browser scroll-into-view <selector>
cmux browser type <selector> <text> [--snapshot-after]
cmux browser fill <selector> [text] [--snapshot-after]   # Empty text clears input
cmux browser press|keydown|keyup <key> [--snapshot-after]
cmux browser select <selector> <value> [--snapshot-after]
cmux browser scroll [--selector <css>] [--dx <n>] [--dy <n>] [--snapshot-after]
```

### Inspection

```bash
cmux browser snapshot [--interactive|-i] [--cursor] [--compact] [--max-depth <n>] [--selector <css>]
cmux browser screenshot [--out <path>] [--json]
cmux browser get <url|title|text|html|value|attr|count|box|styles> [...]
cmux browser is <visible|enabled|checked> <selector>
cmux browser find <role|text|label|placeholder|alt|title|testid|first|last|nth> ...
cmux browser highlight <selector>
```

### JavaScript

```bash
cmux browser eval <script>
cmux browser addinitscript <script>        # Run on every navigation
cmux browser addscript <script>            # Inject script tag
cmux browser addstyle <css>                # Inject stylesheet
```

### Tabs

```bash
cmux browser tab new [url]                 # Open new tab
cmux browser tab list                      # List tabs
cmux browser tab switch <index>            # Switch to tab
cmux browser tab close                     # Close current tab
```

### State & Session

```bash
cmux browser state save <path>             # Save cookies + storage
cmux browser state load <path>             # Restore saved state
cmux browser cookies <get|set|clear> [...]
cmux browser storage <local|session> <get|set|clear> [...]
```

### Frames, Dialogs, Downloads

```bash
cmux browser frame <selector|main>         # Target iframe or return to main
cmux browser dialog <accept|dismiss> [text]
cmux browser download [wait] [--path <path>] [--timeout-ms <ms>]
```

### Console & Errors

```bash
cmux browser console <list|clear>
cmux browser errors <list|clear>
```

### Common Workflow

```bash
# Open site → wait → snapshot → interact → verify
cmux browser open https://example.com
cmux browser wait --load-state complete
cmux browser snapshot -i
# Use refs from snapshot output (e.g. [1] button "Login")
cmux browser click "[1]"
cmux browser wait --text "Welcome"
cmux browser state save ./auth.json
```

## Socket API

Programmatic control via Unix socket at `/tmp/cmux.sock`. Request format: newline-terminated JSON.

| Method | Purpose |
|--------|---------|
| `workspace.list` | List all workspaces |
| `workspace.create` | Create new workspace |
| `workspace.select` | Switch to workspace |
| `surface.send_text` | Send text to terminal |
| `surface.send_key` | Send keypress (enter, tab, escape) |
| `set-status` | Sidebar status pill (icon, color) |
| `set-progress` | Progress bar (0.0-1.0) |
| `notification.create` | Push notification |

CLI flags: `--json`, `--workspace ID`, `--surface ID`, `--id-format refs|uuids|both`

Docs: https://www.cmux.dev/docs/api
