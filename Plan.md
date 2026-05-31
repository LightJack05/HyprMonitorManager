# HyprMonitorManager

A utility for managing setups with multiple monitor configurations on Hyprland.

> [!NOTE]
> This is an opinionated script. Make it simple yet effective. Don't add overly-ambitious configuration options. Just make sure you make it work reliably.

## Technical Decision: Bash

This project is implemented as Bash scripts. Rationale:
- Every operation is calling an external tool (`hyprctl`, `jq`, `ncat`, `zenity`, `nwg-displays`, `notify-send`)
- No complex data structures needed — just files, pipes, and string processing
- File operations (symlinks, config read/write) are Bash's strength
- Socket listening is a simple `ncat` pipe
- The tool is fundamentally glue between existing programs
- Minimal dependencies for Arch users already running Hyprland

**Required tools:** `hyprctl`, `jq`, `ncat` (from nmap), `zenity`, `nwg-displays`, `notify-send`

## Architecture

Single script with subcommands: `hyprmonitormanager`

```
hyprmonitormanager daemon      # Background listener for monitor hotplug events
hyprmonitormanager apply       # Apply config for current monitor set (or launch wizard)
hyprmonitormanager configure   # Force-launch the configuration wizard
hyprmonitormanager list        # List stored configurations
hyprmonitormanager delete ID   # Delete a stored configuration
```

### File Layout

```
~/.config/hyprmonitormanager/
├── settings.conf                    # User settings
└── configs/
    ├── <fingerprint>/
    │   ├── monitors.lua             # hl.monitor({...}) blocks (portable)
    │   └── priorities               # description=priority_number (one per line)
    └── ...
```

The generated output is written to the symlink target (`~/.config/hypr/display.lua` by default), which must already be loaded via `require("display")` in `hyprland.lua`.

## Agent Definitions

### 1. Daemon Agent (background event listener)

**Responsibility:** Listen for monitor hotplug events and trigger config application.

**Implementation:** A `while read` loop on Hyprland's IPC socket2 via `ncat`.

**Behavior:**
- On startup: immediately run `apply` to handle monitors already connected at login
- Listen on `$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock`
- React to `monitoraddedv2>>*` and `monitorremoved>>*` events
- Debounce: on first event, wait 2 seconds, drain queued events, then run `apply`
- Exit cleanly when the socket closes (Hyprland exit) or on SIGTERM/SIGINT

**Skeleton:**
```bash
handle_monitor_change  # Initial check on startup

while IFS= read -r event; do
    case "$event" in
        monitoraddedv2\>*|monitorremoved\>*)
            sleep 2
            while IFS= read -t 0.1 -r _; do :; done  # Drain queue
            handle_monitor_change
            ;;
    esac
done < <(ncat -U "$SOCKET2_PATH")
```

### 2. Fingerprint Agent (monitor set identification)

**Responsibility:** Create a unique, stable identifier for the current set of connected monitors.

**Implementation:** Sort monitor descriptions (make+model+serial from `hyprctl monitors -j`), hash with sha256, truncate to 16 hex chars.

**Why descriptions, not port names:** Port names like `DP-2` change when you use a different port or cable. Descriptions (`ASUSTek COMPUTER INC XG27AQDMG T3LMRS043168`) are tied to the physical display.

```bash
fingerprint() {
    hyprctl monitors -j \
        | jq -r '[.[] | .description] | sort | join("\n")' \
        | sha256sum | cut -c1-16
}
```

### 3. Apply Agent (config generation and activation)

**Responsibility:** Generate a complete hyprland config from stored data, write it to the target path, reload, and move workspaces.

**Key design choice — runtime workspace generation:**
- `hl.monitor({...})` blocks are stored as-is using `desc:` syntax (portable across port changes)
- `hl.workspace_rule({...})` blocks are generated at apply time by resolving descriptions to current port names via `hyprctl monitors -j`
- This means configs survive port reassignment (e.g., switching from DP-2 to DP-3)

**Steps:**
1. Read stored `monitors.lua` (the `hl.monitor({...})` blocks)
2. Read stored `priorities` file
3. For each priority entry, look up the current port name: `hyprctl monitors -j | jq -r --arg d "$desc" '.[] | select(.description==$d) | .name'`
4. Generate `hl.workspace_rule({...})` blocks: priority 1 gets workspaces 1-N, priority 2 gets (N+1)-2N, etc. (N = configurable, default 10). First workspace on each monitor gets `default = true`
5. Write combined output to target path
6. Run `hyprctl reload`
7. Move workspaces to their correct monitors:
   ```bash
   hyprctl -j workspacerules | jq -r '.[] | select(.monitor != "") | "\(.workspaceString) \(.monitor)"' \
       | while read -r ws mon; do
           hyprctl dispatch moveworkspacetomonitor "$ws $mon"
         done
   ```
8. Run post-update hook if configured

### 4. Wizard Agent (interactive configuration)

**Responsibility:** Guide the user through display configuration when no stored config matches.

**UI:** `zenity` for dialogs (Wayland/GTK-native, fast, ESC-dismissible). `nwg-displays` for spatial arrangement.

**Flow:**

1. **Entry point**: called by `apply` when no matching config exists, or directly via `configure` subcommand
2. **Prompt to configure** (skipped if called directly):
   - `zenity --question "New display configuration detected. Configure now?"`
   - If declined: exit (leave Hyprland defaults in place)
3. **Mirror check** (skippable via `SKIP_MIRROR_PROMPT=true` in settings):
   - Only shown when exactly 2 monitors are connected (mirroring 3+ is unusual)
   - `zenity --question "Mirror displays?"`
   - If yes → **Mirror path**
   - If no → **Arrange path**

**Mirror path:**
1. Identify internal vs external display (internal = the one already present before the event, or the laptop panel if detectable)
2. Generate config: `hl.monitor({ output = "desc:EXTERNAL", mode = "preferred", position = "auto", scale = SCALE, mirror = "INTERNAL_PORT" })`
3. Scale optimization: for the mirrored display, calculate scale as `external_width / internal_effective_width`. If the external is lower-res, use `1`; if higher-res, scale up proportionally
4. Store config and apply

**Arrange path:**
1. Launch `nwg-displays` with temp output paths:
   ```bash
   tmpdir=$(mktemp -d)
   nwg-displays -m "$tmpdir/monitors.conf" -w "$tmpdir/workspaces.conf"
   ```
2. Wait for nwg-displays to exit
3. If temp files weren't written (user closed without saving): abort
4. Parse `$tmpdir/monitors.lua` (nwg-displays writes a `.lua` file alongside the `.conf`) — extract monitor settings, convert port names to `desc:` syntax:
   ```bash
   # For each hl.monitor({...}) block, replace output = "PORT" with output = "desc:DESCRIPTION"
   while IFS= read -r line; do
       port=$(echo "$line" | grep -oP '(?<=output = ")[^"]+')
       desc=$(hyprctl monitors -j | jq -r --arg n "$port" '.[] | select(.name==$n) | .description')
       echo "$line" | sed "s/output = \"$port\"/output = \"desc:$desc\"/"
   done < "$tmpdir/monitors.lua"
   ```
5. **Priority assignment** — show a zenity dialog per priority level:
   - For 2 monitors: single dialog "Select the primary display" (other gets priority 2)
   - For 3+: sequential dialogs, each showing remaining unassigned monitors
   - Display labels: `MAKE MODEL (RESxRES)` for clarity
6. Save `monitors.lua` and `priorities` to config dir
7. Apply

### 5. Settings Agent (configuration)

**Responsibility:** Read user settings, provide defaults.

**File:** `~/.config/hyprmonitormanager/settings.conf` (simple `KEY=value` format, sourced by Bash)

```bash
# Directory for stored monitor configurations
CONFIG_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/hyprmonitormanager/configs"

# Target path for the generated hyprland config (must be loaded via require("display") in hyprland.lua)
TARGET_PATH="$HOME/.config/hypr/display.lua"

# Workspaces allocated per monitor
WORKSPACES_PER_MONITOR=10

# Skip the "mirror displays?" prompt (true/false)
SKIP_MIRROR_PROMPT=false

# Auto-apply known configurations without prompting (true/false)
AUTO_APPLY=true

# Command to run after config is applied (e.g., restart waybar)
POST_UPDATE_HOOK=""
```

**Loading:** Source defaults, then overlay user settings if the file exists:
```bash
load_settings() {
    # Defaults (defined inline)
    CONFIG_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/hyprmonitormanager/configs"
    TARGET_PATH="$HOME/.config/hypr/display.lua"
    WORKSPACES_PER_MONITOR=10
    SKIP_MIRROR_PROMPT=false
    AUTO_APPLY=true
    POST_UPDATE_HOOK=""

    local settings_file="${XDG_CONFIG_HOME:-$HOME/.config}/hyprmonitormanager/settings.conf"
    [[ -f "$settings_file" ]] && source "$settings_file"
}
```

## Control Flow

### On Monitor Change (daemon event or manual `apply`)

```
┌─────────────────────────────┐
│   Monitor event detected    │
│   (or manual `apply` call)  │
└────────────┬────────────────┘
             │
             ▼
   ┌──── Fingerprint current monitors ────┐
   │                                      │
   ▼                                      │
Config exists for fingerprint?            │
   │                                      │
   ├── YES ──► Apply stored config ──► Done
   │
   ├── NO + AUTO_APPLY=true ──► Launch wizard
   │
   └── NO + AUTO_APPLY=false ──► Notify user
         ("New display detected. Run hyprmonitormanager configure")
```

### Wizard Flow

```
┌─────────────────────────┐
│  "Configure now?" dialog │ (skipped if called via `configure`)
└────────┬────────────────┘
         │ Yes
         ▼
  2 monitors + mirror prompt enabled?
         │
    ┌────┴────┐
    │ Yes     │ No
    ▼         ▼
"Mirror?"   Go to Arrange
    │
  ┌─┴──┐
  Yes   No
  │     │
  ▼     ▼
Mirror  Arrange
config  ┌────────────────────┐
  │     │ Launch nwg-displays│
  │     │ User arranges      │
  │     │ nwg-displays exits │
  │     └────────┬───────────┘
  │              │
  │     ┌────────▼───────────┐
  │     │ Priority selection  │
  │     │ (zenity dialogs)    │
  │     └────────┬───────────┘
  │              │
  └──────┬───────┘
         ▼
  Save config + Apply
```

## Edge Cases

1. **Docking station (multiple monitors at once):** Debounce handles this — 2-second wait after first event drains all subsequent events before acting.

2. **Startup with multiple monitors:** Daemon runs `apply` immediately on start, before entering the event loop.

3. **Monitor on different port:** Using `desc:` syntax in `hl.monitor({...})` blocks and runtime port-name resolution for `hl.workspace_rule({...})` blocks handles this transparently.

4. **User cancels wizard (ESC/close):** All zenity dialogs and nwg-displays return non-zero on cancel. The wizard exits cleanly, leaving the current config untouched.

5. **Single monitor (no external):** If only one monitor is detected and a config exists, apply it. If not, generate a trivial config (just `hl.monitor({ output = "desc:...", mode = "preferred", position = "auto", scale = SCALE })`). No wizard needed.

6. **nwg-displays closed without saving:** Check if temp output files exist after nwg-displays exits. If not, abort with notification.

7. **Socket disconnection:** When Hyprland exits, the socket closes, `ncat` terminates, the read loop ends, and the daemon exits naturally. No restart logic needed — the next Hyprland session will re-run `hl.exec_once`.

## Integration with Hyprland

The daemon is started via Hyprland's `exec_once` function. The user adds this to their `hyprland.lua`:

```lua
hl.exec_once({ cmd = "hyprmonitormanager daemon" })
```

**Lifecycle:**
- Hyprland starts → `hl.exec_once` launches the daemon
- Daemon runs `apply` immediately (handles monitors already connected at login)
- Daemon listens for hotplug events via socket2
- Hyprland exits → socket closes → `ncat` exits → read loop ends → daemon exits

No systemd service needed. The process is a direct child of Hyprland and dies with it.

**Signal handling:** Trap `SIGTERM` and `SIGINT` to clean up any temp files before exit:
```bash
cleanup() {
    [[ -d "$TMPDIR_WIZARD" ]] && rm -rf "$TMPDIR_WIZARD"
    exit 0
}
trap cleanup SIGTERM SIGINT
```

## Packaging

### PKGBUILD

```bash
pkgname=hyprmonitormanager
pkgver=1.0.0
pkgrel=1
pkgdesc="Monitor configuration manager for Hyprland"
arch=('any')
license=('MIT')
depends=('hyprland' 'jq' 'nmap' 'zenity' 'nwg-displays' 'libnotify')
# nmap provides ncat, libnotify provides notify-send

install -Dm755 hyprmonitormanager "$pkgdir/usr/bin/hyprmonitormanager"
install -Dm644 settings.conf.example "$pkgdir/usr/share/hyprmonitormanager/settings.conf.example"
```

## Implementation Order

1. **Settings + fingerprint** — foundational, everything depends on these
2. **Apply agent** — config generation, symlink, reload, workspace move
3. **Wizard agent** — mirror path, arrange path (nwg-displays + priority dialogs)
4. **Daemon agent** — socket listener, debounce, ties everything together
5. **Subcommands + CLI** — `list`, `delete`, argument parsing, help text
6. **PKGBUILD** — packaging
