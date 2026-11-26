# Better Desktop

A coherent desktop built around [niri](https://github.com/niri-wm/niri): one
scrollable-tiling compositor and a set of small tools that share its IPC,
configuration and release cycle. Each tool is its own Cargo workspace and its
own repository, so it can be installed, tested and released on its own. The
`better` command is only a selector over them.

The packaging lives in [better-niri](https://github.com/fram446742/better-niri):
`sources`, `packaging/`, the Arch/AUR recipes and the documentation that covers
architecture, testing and provenance.

## Repositories

| Repository | What it does |
| --- | --- |
| [better-niri](https://github.com/fram446742/better-niri) | Compositor, IPC/config, packaging and ecosystem documentation |
| [better-smithay](https://github.com/fram446742/better-smithay) | Smithay fork used by the compositor |
| [better-companion](https://github.com/fram446742/better-companion) | Automation daemon and `companionctl`: rules, run-or-raise, staging, scratchpads, swallowing, marks, idle and rotation |
| [better-session](https://github.com/fram446742/better-session) | Named work contexts: workspaces, windows, outputs, hooks |
| [better-monitor](https://github.com/fram446742/better-monitor) | Output profiles, matching rules and safe apply/rollback |
| [better-settings](https://github.com/fram446742/better-settings) | GTK4 control center and KDL-preserving CLI |
| [better-shot](https://github.com/fram446742/better-shot) | Screenshots, recording, annotations, OCR, pins and history |
| [better-audio](https://github.com/fram446742/better-audio) | PipeWire routing by application, window and context |
| [better-wallpaper](https://github.com/fram446742/better-wallpaper) | Scenes, per-workspace wallpapers, video and transitions |
| [better-sidepanels](https://github.com/fram446742/better-sidepanels) | Layer-shell panels and floating side stacks |
| [better-portal](https://github.com/fram446742/better-portal) | Standalone Screenshot, PickColor and ScreenCast portal |
| [better-noctalia-plugins](https://github.com/fram446742/better-noctalia-plugins) | Noctalia bar widgets for every tool |
| [better-nsticky](https://github.com/fram446742/better-nsticky) | External sticky/staged-window workflow that companion absorbs |

## Install

On Arch, use the split package base in `better-niri/packaging/aur/`. The
`better-desktop` meta package installs the compositor, every companion, the
Noctalia plugin source and the `better` selector:

```sh
paru -S better-desktop
```

The individual packages (`better-shot`, `better-settings`, `better-session`, …)
can be installed without the compositor. They only need a running niri session
through `$NIRI_SOCKET`, plus their own runtime dependencies.

From source, build the whole set with the compositor repository:

```sh
git clone https://github.com/fram446742/better-niri
cd better-niri
./scripts/build-dist.sh
```

Each tool also builds alone:

```sh
git clone https://github.com/fram446742/better-shot
cd better-shot
cargo build --release
```

Tools that need niri's IPC depend on the `niri-ipc` crate from the better-niri
workspace, pinned to `integration/elite-base`.

## Use

The selector keeps the commands in one place. It does not own state or policy;
it runs the installed binary, or falls back to the tool's own workspace in a
source checkout.

```sh
better --list
better shot screen --copy
better shot ui
better monitor list
better monitor apply desk
better session save work --switch
better session restore work
better audio sinks
better audio apply
better wallpaper status
better wallpaper apply
better sidepanels --check
better companion status
better settings
```

The individual binaries remain the authority:

```sh
companionctl call windows run-or-raise '{"app_id":"firefox","spawn":["firefox"]}'
companionctl call rules set '{"name":"term","match":{"app_id":"Alacritty"},"steps":[…] }'
better-monitor watch
better-session list
better-shot history list
better-wallpaper watch
better-audio rules apply
better-settingsctl config dump
```

A new niri session usually starts these from `~/.config/niri/config.kdl`:

```kdl
spawn-at-startup "better-portal"
spawn-at-startup "better-companion"
spawn-at-startup "better-wallpaper" "watch"
spawn-at-startup "better-sidepanels"
```

Enable the portal backend in
`~/.config/xdg-desktop-portal/portals.conf` only when `better-portal` is
running; otherwise keep the compositor's own portal.

## How the pieces fit

- **Compositor**: niri with the merged community work. It owns seats, outputs,
  layout, capture and the privileged protocols.
- **Companion**: one daemon, one IPC connection, internal modules for policy.
  It never opens a second niri socket and never becomes a compositor plugin.
- **Tools**: separate processes with versioned JSON or D-Bus interfaces. They
  are allowed to fail without taking the session down.
- **Shell**: Noctalia, connected through Luau widgets that call the public CLIs.
  No second shell or bar is shipped.

This is why the ecosystem is split by responsibility rather than by product:
changes in `better-shot` do not rebuild the compositor, and a broken rule in
`better-companion` cannot corrupt the render loop. The boundaries and state files
are described in [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## HDR and display limits

The compositor implements the HDR scanout pipeline: per-surface descriptions,
sRGB→PQ blending, reference-white handling, post-blend PQ encode, HDR cursor
columns and HDR capture paths. The KMS side still needs validation on NVIDIA
hardware; the test suite covers the shader/CPU math and RADV rendering.

- **Output mirroring** is not a compositor primitive in niri. `better-monitor`
  runs `wl-mirror <source> --fullscreen-output <target>` for profile mirror
  relations and tracks only its own children; a native mirror would need an
  output-composition feature.
- **HDR wallpapers** now have a real path for HDR video: `better-wallpaper`
  submits 10-bit PQ/BT.2020 `wl_shm` frames with a `wp_color_manager_v1`
  description when the output is HDR-active, and falls back to SDR tone-mapping
  otherwise. Static images stay SDR. This still needs a live HDR/Wayland run;
  the offline tests cover the pixel math, format selection and fallbacks.

See the [HDR notes](https://github.com/fram446742/better-niri/blob/integration/elite-base/docs/HDR.md)
and `better-niri/docs/UNIMPLEMENTED.md` for the exact remaining list and the
COSMIC comparison.

## AUR

The split recipes live in `better-niri/packaging/aur/`:

- `niri-better` (pinned source, [published on AUR](https://aur.archlinux.org/packages/niri-better)) owns every split package.
- `niri-better-git` (branch recipe) is kept for local builds; AUR rejects it
  alongside `niri-better` because the split package names would collide.

The package base owns all binaries, including `better`. AUR upload steps and
the package list are in [`docs/AUR.md`](docs/AUR.md).

## Provenance

Better Desktop would not exist without niri, Smithay and the community forks and
tools it builds on. The compositor and companion trees record the exact sources
in their `docs/ATTRIBUTION.md` and `docs/ECOSYSTEM_RESEARCH.md`. The short list:

niri, niri-elite, biri, SHORiN, urayde, niri-spicy, Naxdy, demoji, Niri-glass,
the Smithay lineage, Piri, nsticky, niri-sidepanels, nirinit,
niri-session-manager, NiriMod, niri_display_manager, PWSW, iio-niri, stasis,
niri-cwd, niri-screen-time, Nirisnap, niri-shot, Noctalia and its plugin
catalog.

Everything is GPL-3.0-or-later unless a linked repository says otherwise;
Smithay-derived code keeps its MIT notices.
