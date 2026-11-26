# AUR publishing

The VCS recipe is
[`loom-niri/packaging/aur/loom-niri-git`](https://github.com/fram446742/loom-niri/tree/main/packaging/aur/loom-niri-git).
It is a split package base: one checkout builds the compositor and every tool.

## Package base

| AUR repository | Packages |
| --- | --- |
| `loom-niri-git` (published) | `loom-niri-git`, `loom-niri-replace-git`, `loom-portal-git`, `loom-companion-git`, `loom-monitor-git`, `loom-session-git`, `loom-shot-git`, `loom-wallpaper-git`, `loom-settings-git`, `loom-audio-git`, `loom-sidepanels-git`, `loom-tui-git`, `loom-noctalia-plugins-git`, `loom-desktop-git` |

`loom-niri-git` installs `loom-niri` with its own XDG paths and does not
conflict with niri. `loom-niri-replace-git` installs the `niri` names and
conflicts with `niri`/`niri-git`. The two can be installed together but only one
can own a DRM session.

A later stable set (`loom-niri`, `loom-shot`, …) and `-bin` set use the same
recipes without the `-git` suffix. No package named just `loom` is possible:
that AUR name is already taken by another project; the umbrella repository is
`github.com/fram446742/loom`.

## Before uploading

1. `packaging/scripts/verify.sh` must pass.
2. `packaging/scripts/update-srcinfo.sh` refreshes both the stable and VCS
   `.SRCINFO` files.
3. Copy `PKGBUILD`, `.SRCINFO`, `loom-desktop-git.install`,
   `loom-desktop-git.README.md` and `loom-settings.desktop` into the AUR
   checkout.

## Upload

```sh
git clone ssh://aur@aur.archlinux.org/loom-niri-git.git
cd loom-niri-git
# copy the recipe files
git add PKGBUILD .SRCINFO loom-desktop-git.install \
  loom-desktop-git.README.md loom-settings.desktop
git commit -m "Update to $(grep ^pkgver= PKGBUILD | cut -d= -f2)"
git push
```

Only `loom-niri-git` owns these package names. The old `niri-better` base is
retired and can be deleted from the AUR.

## Release checklist

```sh
cd loom-niri
./packaging/scripts/verify.sh
cargo test --workspace
cargo test -p niri --no-default-features
./scripts/test-all.sh
./packaging/scripts/update-srcinfo.sh
```

Then push the AUR repository and tag the source releases.
