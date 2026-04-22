# HyprMonitorManager

A utility for managing setups with multiple monitor configurations on Hyprland.

> [!NOTE]
> This is an opinionated script. Make it simple yet effective. Don't add overly-ambitions configuration options. Just make sure you make it work reliably.

## Control Flow
- Listen for monitor events (HDMI, DP, USB-C-DP, etc.), for example via udev.
- On Monitor Plugin:
    - Check if the current configuration already has a hyprland config file in the directory. 
        - If it does:
            - Symlink it to ~/.config/hypr/display.conf and `hyprctl reload`
            - EXIT
        - If it does not, ask the user wether to configure displays, if yes:
            - Ask (via GUI) wether to mirror the display (This check can be configured to be skipped in settings).
                - If mirroring:
                    - Set the external display to mirror the internal one. 
                    - Optimize scaling and resolution for the external display. (Note that you may want different scales on the monitors, such as running 2x on the internal one and 1x on the external one. Figure out what would be best)
                    - EXIT
                - If not mirroring:
                    - Open up a GUI configuration utility (check if there is a better one than nwg-displays, or if that is the golden standard) to allow the user to arrange the displays.
                    - Read the new configuration, including mirroring and resolution options etc. Discard workspace assignments if present.
                    - Ask which priority the displays hold (starting with 1 up to the number of displays. No duplicates, no left out ones.)
                    - Based on priority, assign the monitor a configurable number of workspaces (default 10, so 1-10 for priority 1, 11-20 for priority 2, etc.)
                    - Store the configuration in a configurable directory
                    - Symlink the config to ~/.config/hypr/display.conf and `hyprctl reload`


## Packaging
Package this via a PKGBUILD

## Implementation constraints
Keep the UI fast and easily dismissed. As far as possible, ESC anywhere just exits the config program when the window is in focus.
Use a UI that works well with hyprland and wayland.
Make at least the display config directory, the target path for the symlink, and a post-update hook configurable.

If the display config has changed, move the workspaces to the correct monitor with a script kind of like this:

```
#!/usr/bin/env zsh

hyprctl -j workspacerules | jq -r '.[] | select(.monitor != "") | "\(.workspaceString) \(.monitor)"' | while read -r workspace monitor; do
    # Dispatch the move command for each rule found
    hyprctl dispatch moveworkspacetomonitor "$workspace $monitor"
done
```
