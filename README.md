# Better Displays

A [Omarchy](https://omarchy.org/) shell plugin that puts **granular display and
terminal control** in a bar widget you open directly from the status bar.

![preview](preview.png)

## Features

- **Per-monitor** resolution (real modes reported by the monitor), scale,
  position (left/right/above/below another display), and orientation
  (0°/90°/180°/270°).
- **Per-terminal font size** for alacritty, kitty, ghostty, and foot, with
  live − / + steppers.
- All changes apply **live** via Hyprland and **persist** to
  `~/.config/omarchy/displays.json` and `~/.config/hypr/monitors.lua` (so they
  survive a reboot).
- Backend scripts placed on `PATH` (`omarchy-display-monitor`,
  `omarchy-display-terminal`, `omarchy-display-pick`) for scripting and for
  wiring into the Omarchy menu.

## Requirements

- Omarchy (Hyprland-based) with the Quickshell shell.
- `bash` and `jq` (both standard on Omarchy).
- `hyprctl` (provided by Hyprland).

## Install

### Via `omarchy plugin add` (recommended)

```bash
omarchy plugin add https://github.com/nightdevil00/better.displays.git --enable
```

This clones the plugin, validates it, and enables the bar widget. When the
shell loads the plugin it **auto-installs the backend scripts** onto `PATH`
(`~/.local/bin`, falling back to `/usr/local/bin`), so they can be called
directly or from a menu entry. No manual step needed.

Note that `omarchy display ...` will **not** dispatch to these scripts. The
Omarchy CLI resolves a group's subcommands by globbing its own installation
directory, not `PATH`, so commands shipped by a plugin are never discovered
there. Call the scripts by name instead.

### Manual

```bash
git clone https://github.com/nightdevil00/better.displays.git \
  ~/.config/omarchy/plugins/better.displays
cd ~/.config/omarchy/plugins/better.displays
./install                 # symlink the backend scripts into ~/.local/bin
omarchy plugin enable better.displays
omarchy restart shell
```

## Use it

- Click the **Better Displays** icon in the bar (next to the monitor icon), or
  summon it: `omarchy-shell shell summon better.displays`.
- Pick a monitor, then adjust Resolution / Scale / Position / Orientation, and
  tune each terminal's font size.

Equivalents from a shell:

```bash
omarchy-display-monitor list
omarchy-display-monitor set DP-1 --mode 2560x1440@144 --scale 1.6 --pos 0x0 --transform 0
omarchy-display-terminal list
omarchy-display-terminal set ghostty 14
omarchy-display-terminal set-all 13
```

To reach the same pickers from the Omarchy menu, add entries to
`~/.config/omarchy/extensions/omarchy-menu.jsonc`:

```jsonc
"setup.displays": {"icon":"\udb81\udf79","label":"Displays","when":"command -v omarchy-display-pick >/dev/null"},
"setup.displays.mode": {"icon":"\udb81\udf79","label":"Resolution","action":"omarchy-display-pick monitor mode"},
"setup.displays.scale": {"icon":"\udb82\udc78","label":"Scale","action":"omarchy-display-pick monitor scale"},
"setup.displays.position": {"icon":"\udb82\ude72","label":"Position","action":"omarchy-display-pick monitor position"},
"setup.displays.transform": {"icon":"\udb80\udd3a","label":"Orientation","action":"omarchy-display-pick monitor transform"},
"setup.displays.terminal": {"icon":"\udb84\udc98","label":"Terminal font size","action":"omarchy-display-pick terminal"},
```

## Uninstall

```bash
~/.config/omarchy/plugins/better.displays/uninstall   # drop the PATH symlinks
omarchy plugin remove better.displays
```

(`uninstall` removes the `omarchy-display-*` symlinks; `omarchy plugin remove`
deletes the plugin folder. Order does not matter, but run both for a clean
removal.)

## How it works

The plugin folder bundles everything it needs:

```
better.displays/
├── manifest.json        # plugin metadata (bar-widget)
├── Panel.qml            # the bar widget + popup (reuses Omarchy's qs.Ui kit)
├── bin/                 # backend scripts (self-contained, travel with the plugin)
│   ├── omarchy-display-monitor
│   ├── omarchy-display-terminal
│   └── omarchy-display-pick
├── install              # symlink bin/* into ~/.local/bin (idempotent)
├── uninstall            # remove those symlinks
├── preview.png
└── README.md
```

`Panel.qml` invokes the scripts by their absolute path inside `bin/`, so the
widget works the moment the folder is present — the `install` step only exists
to put the scripts on `PATH` for shell and menu use.

## Security

The plugin receives monitor names, mode strings, positions, and scale values
from Hyprland (`hyprctl monitors -j`) and passes them through several layers:

**Panel.qml** — constructs `bash -c` commands to call the backend scripts. All
interpolated values (monitor name, flags, terminal names, sizes) are wrapped
with `shellEscape()` which quotes each argument with single quotes and escapes
any embedded single quotes, preventing shell metacharacter injection.

**omarchy-display-monitor** —

| Concern | Mitigation |
| --- | --- |
| Monitor name in jq filter | `jq --arg` used instead of string interpolation, so names cannot break out of the filter expression |
| Values embedded in Lua expressions (`hyprctl eval`, `monitors.lua`) | All inputs validated against strict regex patterns **before** use: output names match `[a-zA-Z0-9_-]+(:[a-zA-Z0-9_-]+)?`, modes match `WxH@R[Hz]`, positions match `XxY` or `auto`, scales are numeric, transforms are `0`–`3`. String values are also run through `lua_escape()` which escapes `\` and `"` for safe Lua double-quote embedding |
| Values used in grep/awk patterns | `persist_to_lua` uses the same `lua_escape()` output in its grep regex and awk `-v` assignments |

**omarchy-display-terminal** — validates that the terminal name is one of the
known set (`alacritty`, `kitty`, `ghostty`, `foot`) via a whitelist check and
that the font size is numeric (`^[0-9]+(\.[0-9]+)?$`).

## License

MIT — do what you like, attribute if you're feeling generous.
