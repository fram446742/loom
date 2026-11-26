# Loom

A connected ecosystem for Linux: one scrollable-tiling compositor and a set of
independent tools that share niri's IPC and configuration. Each tool is its own
Cargo workspace and repository, so it can be installed, tested and released on
its own. The `loom` command is only a selector over them.

Loom is built around niri. The compositor keeps the `niri` name where it needs
to, and adds a coexisting `loom-niri` identity with its own config, socket,
portal bus and session files.

## Repositories

| Repository | What it does |
| --- | --- |
| [loom](https://github.com/fram446742/loom) | This index, usage and ecosystem documentation |
| [loom-niri](https://github.com/fram446742/loom-niri) | Compositor, IPC/config, `loom-state` and packaging |
| [loom-smithay](https://github.com/fram446742/loom-smithay) | Smithay fork used by the compositor |
| [loom-companion](https://github.com/fram446742/loom-companion) | Automation daemon and `loomctl`: rules, run-or-raise, staging, scratchpads, swallowing, marks, idle and rotation |
| [loom-session](https://github.com/fram446742/loom-session) | Named work contexts: workspaces, windows, outputs and hooks |
| [loom-monitor](https://github.com/fram446742/loom-monitor) | Output profiles, matching rules and safe apply/rollback |
| [loom-settings](https://github.com/fram446742/loom-settings) | GTK4 control center and KDL-preserving CLI |
| [loom-shot](https://github.com/fram446742/loom-shot) | Screenshots, recording, annotations, OCR, pins and history |
| [loom-audio](https://github.com/fram446742/loom-audio) | PipeWire routing by application, window and context |
| [loom-wallpaper](https://github.com/fram446742/loom-wallpaper) | Scenes, per-workspace wallpapers, HDR video and transitions |
| [loom-sidepanels](https://github.com/fram446742/loom-sidepanels) | Layer-shell panels, floating side stacks and the optional minimap |
| [loom-tui](https://github.com/fram446742/loom-tui) | Terminal dashboard for outputs, workspaces and windows |
| [loom-portal](https://github.com/fram446742/loom-portal) | Standalone Screenshot, PickColor and ScreenCast portal |
| [loom-noctalia-plugins](https://github.com/fram446742/loom-noctalia-plugins) | Noctalia bar widgets for every tool |
| [loom-nsticky](https://github.com/fram446742/loom-nsticky) | External sticky/staged-window workflow that companion absorbs |

## Install

On Arch, the AUR package base `loom-niri-git` produces two compositor variants:

- `loom-niri-git` installs `loom-niri`, with its own XDG paths. It does not
  conflict with niri and is the safe way to test Loom next to an existing niri.
- `loom-niri-replace-git` installs `niri`, `niri-session`, the niri portal and
  session unit, and conflicts with `niri`/`niri-git`.

The rest of the split set is `loom-portal-git`, `loom-companion-git`,
`loom-monitor-git`, `loom-session-git`, `loom-shot-git`, `loom-wallpaper-git`,
`loom-settings-git`, `loom-audio-git`, `loom-sidepanels-git`,
`loom-tui-git`, `loom-noctalia-plugins-git` and the `loom-desktop-git` meta
package.

```sh
paru -S loom-niri-git          # coexist with niri
paru -S loom-niri-replace-git  # replace niri
paru -S loom-desktop-git       # compositor + tools
```

A stable source set (`loom-niri`, `loom-shot`, …) and prebuilt `-bin` packages
can follow from the same recipes later.

From source, clone the compositor and build the tools:

```sh
git clone https://github.com/fram446742/loom-niri
cd loom-niri
./scripts/build-dist.sh
```

Each tool also builds alone:

```sh
git clone https://github.com/fram446742/loom-shot
cd loom-shot
cargo build --release
```

Tools that need niri's IPC depend on the `niri-ipc` and `loom-state` crates from
the `loom-niri` workspace.

## Use

The selector keeps the commands in one place. It runs the installed binary, or
falls back to the tool's own workspace in a source checkout.

```sh
loom --list
loom tui
loom shot screen --copy
loom shot ui
loom monitor list
loom monitor apply desk
loom session save work --switch
loom session restore work
loom audio sinks
loom audio apply
loom wallpaper status
loom wallpaper apply
loom sidepanels --check
loom companion status
loom settings
```

The individual binaries remain the authority:

```sh
loomctl call windows run-or-raise '{"app_id":"firefox","spawn":["firefox"]}'
loomctl call rules set '{"name":"term","match":{"app_id":"Alacritty"},"steps":[…] }'
loom-monitor watch
loom-session list
loom-shot history list
loom-wallpaper watch
loom-audio rules apply
loom-settingsctl config dump
loom-tui
```

A Loom session usually starts these from its config:

```kdl
spawn-at-startup "loom-portal"
spawn-at-startup "loom-companion"
spawn-at-startup "loom-wallpaper" "watch"
spawn-at-startup "loom-sidepanels"
```

The built-in portal is `loom-niri.portal`; the standalone backend is
`loom.portal`. Select the standalone backend in
`~/.config/xdg-desktop-portal/portals.conf` only when `loom-portal` is running.

## How the pieces fit

- **Compositor**: niri with the merged community work. It owns seats, outputs,
  layout, capture and the privileged protocols. `loom-niri` and the replacement
  package are the same binary with a different runtime identity.
- **State**: `loom-state` is a small library that owns one event stream and a
  cache. Sidepanels, the minimap and the TUI read it instead of polling niri.
- **Companion**: one daemon, one IPC connection, internal modules for policy.
- **Tools**: separate processes with versioned JSON or D-Bus interfaces. They
  are allowed to fail without taking the session down.
- **Shell**: Noctalia, connected through Luau widgets that call the public CLIs.

The boundaries and state files are described in
[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## HDR and display limits

The compositor implements the HDR scanout pipeline: per-surface descriptions,
sRGB→PQ blending, reference-white handling, post-blend PQ encode, HDR cursor
columns and HDR capture paths. The KMS side still needs validation on NVIDIA
hardware; the test suite covers the shader/CPU math and RADV rendering.

- **Output mirroring** is not a compositor primitive. `loom-monitor` runs
  `wl-mirror <source> --fullscreen-output <target>` for profile mirror
  relations and tracks only its own children.
- **HDR wallpapers** have a real path for HDR video: `loom-wallpaper` submits
  10-bit PQ/BT.2020 `wl_shm` frames with a `wp_color_manager_v1` description
  when the output is HDR-active, and falls back to SDR tone-mapping otherwise.
  Static images stay SDR. This still needs a live HDR/Wayland run.

See the [HDR notes](https://github.com/fram446742/loom-niri/blob/main/docs/HDR.md)
for the COSMIC comparison and the exact remaining checks.

## AUR

The VCS recipe lives in `loom-niri/packaging/aur/loom-niri-git/` and is
published at `ssh://aur@aur.archlinux.org/loom-niri-git.git`. The old
`niri-better` base is retired and can be deleted from the AUR.

The package base owns all binaries, including `loom`. Release steps and the
full package list are in [`docs/AUR.md`](docs/AUR.md).

## Provenance

Loom would not exist without niri, Smithay and the community forks and tools it
builds on. The compositor and companion trees record the exact sources in
`docs/ATTRIBUTION.md` and `docs/ECOSYSTEM_RESEARCH.md`. The short list:

niri, niri-elite, biri, SHORiN, urayde, niri-spicy, Naxdy, demoji, Niri-glass,
the Smithay lineage, Piri, nsticky, niri-sidepanels, nirinit,
niri-session-manager, NiriMod, niri_display_manager, PWSW, iio-niri, stasis,
niri-cwd, niri-screen-time, Nirisnap, niri-shot, hyprlax and Noctalia.

Everything is GPL-3.0-or-later unless a linked repository says otherwise;
Smithay-derived code keeps its MIT notices.
