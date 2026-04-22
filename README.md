# HyprMonitorManager

A monitor hotplug manager for Hyprland. Automatically applies saved display configurations when you connect a known set of monitors, and walks you through setup when you connect something new.

## How it works

The daemon listens on Hyprland's IPC socket for monitor connect/disconnect events. When the set of connected monitors changes:

1. It fingerprints the current monitors (using their hardware descriptions — stable across port changes)
2. If a saved config exists for that set: applies it immediately, silently
3. If no config exists: prompts you to configure the new setup via a short wizard

The wizard uses `nwg-displays` for spatial arrangement and `zenity` dialogs for everything else. When done, the config is saved and applied. The next time you connect that same set of monitors, step 1 handles everything automatically.

## Dependencies

| Package | Purpose |
|---------|---------|
| `hyprland` | Required — uses `hyprctl` and IPC socket |
| `jq` | JSON parsing of `hyprctl` output |
| `nmap` | Provides `ncat` for socket communication |
| `zenity` | GTK dialogs for the wizard |
| `nwg-displays` | Display arrangement GUI |
| `libnotify` | Provides `notify-send` for notifications |

On Arch: `pacman -S jq nmap zenity nwg-displays libnotify`

## Installation

### From AUR / PKGBUILD

```bash
git clone https://github.com/LightJack05/HyprMonitorManager
cd HyprMonitorManager
makepkg -si
```

### Manual

```bash
sudo install -Dm755 hyprmonitormanager /usr/local/bin/hyprmonitormanager
```

## Setup

### 1. Ensure `display.conf` is sourced in your Hyprland config

Add this to `~/.config/hypr/hyprland.conf` if it isn't already there:

```
source = ~/.config/hypr/display.conf
```

Create an empty file so Hyprland doesn't error on first launch before any config is saved:

```bash
touch ~/.config/hypr/display.conf
```

### 2. Start the daemon via exec-once

Add to `~/.config/hypr/hyprland.conf`:

```
exec-once = hyprmonitormanager daemon
```

The daemon starts with Hyprland, handles monitors already connected at login, then listens for hotplug events. It exits naturally when Hyprland exits — no separate process management needed.

### 3. (Optional) Create a settings file

```bash
cp /usr/share/hyprmonitormanager/settings.conf.example \
   ~/.config/hyprmonitormanager/settings.conf
```

Edit as needed. All settings have defaults so the file is optional.

## First use

On first launch with a new monitor set, a dialog appears asking if you want to configure the displays. If you say yes:

**With 2 monitors:** you're asked whether to mirror or arrange independently.

**Mirroring:** the external display mirrors the internal one (useful for presentations). The config is saved automatically.

**Arranging:** `nwg-displays` opens so you can drag monitors into position, set resolutions, and adjust scale. After you save and close it, you're asked to assign a priority to each display. Priority determines workspace assignment:

- Priority 1 → workspaces 1–10
- Priority 2 → workspaces 11–20
- Priority 3 → workspaces 21–30
- (configurable via `WORKSPACES_PER_MONITOR`)

After completing the wizard, the config is saved and applied. Future connections of the same monitor set skip the wizard entirely.

**With 3+ monitors:** the mirror prompt is skipped and you go straight to arrangement.

## Commands

```
hyprmonitormanager daemon       Start the hotplug listener (use exec-once)
hyprmonitormanager apply        Apply the saved config for current monitors
hyprmonitormanager configure    Run the wizard for the current monitor set
hyprmonitormanager list         Show all saved configurations
hyprmonitormanager delete <id>  Delete a saved configuration
```

`configure` is useful to reconfigure an existing setup — it skips the "configure now?" prompt and goes directly into the wizard, overwriting the previously saved config for the current monitor set.

## Settings

Settings file location: `~/.config/hyprmonitormanager/settings.conf`

| Setting | Default | Description |
|---------|---------|-------------|
| `CONFIG_DIR` | `~/.config/hyprmonitormanager/configs` | Where configurations are stored |
| `TARGET_PATH` | `~/.config/hypr/display.conf` | Output file (must be sourced in hyprland.conf) |
| `WORKSPACES_PER_MONITOR` | `10` | Workspaces allocated per display |
| `SKIP_MIRROR_PROMPT` | `false` | Skip the "mirror displays?" dialog |
| `AUTO_APPLY` | `true` | Launch wizard on unknown monitor set. Set to `false` to get a notification instead |
| `POST_UPDATE_HOOK` | _(empty)_ | Shell command run after each config apply (e.g. `pkill -SIGUSR2 waybar`) |

## How configurations are stored

Each unique monitor set gets its own directory under `CONFIG_DIR`, named by a 16-character fingerprint derived from the hardware descriptions of the connected displays. This fingerprint is stable: it's based on the monitor's make, model, and serial number — not the port it's plugged into. Plugging the same monitor into a different DP port still matches the saved config.

```
~/.config/hyprmonitormanager/configs/
└── a3f1c8e09b2d4517/
    ├── monitors.conf   # monitor=desc:... lines
    └── priorities      # display descriptions, ordered by priority
```

`monitors.conf` uses Hyprland's `desc:` monitor syntax, which targets displays by description rather than port name. Workspace rules are regenerated fresh on each apply, so they always reference the correct current port names.

## Reconfiguring a setup

To change the configuration for a monitor set you've already saved:

1. Connect that set of monitors
2. Run `hyprmonitormanager configure`

This launches the wizard and overwrites the existing config for that set. Alternatively, delete the config with `hyprmonitormanager delete <id>` (find the id via `hyprmonitormanager list`) and reconnect the monitors to trigger the wizard automatically.

## Troubleshooting

**The daemon isn't starting:**
Make sure `HYPRLAND_INSTANCE_SIGNATURE` is set in your environment. This is set automatically by Hyprland for processes launched via `exec-once`.

**Config is applied but workspaces aren't on the right monitors:**
Hyprland's workspace rules require the display to be connected. If a workspace was manually moved before the config applied, run `hyprmonitormanager apply` to re-trigger the workspace move.

**nwg-displays closes but no config is saved:**
nwg-displays must be closed via its own UI (not killed). If it exits without writing output files, the wizard treats this as a cancellation and leaves the current config untouched.

**I want to remove all saved configurations:**
```bash
rm -rf ~/.config/hyprmonitormanager/configs/
```
