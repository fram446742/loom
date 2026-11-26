# AUR publishing

The source recipes are maintained in
[`better-niri/packaging/aur`](https://github.com/fram446742/better-niri/tree/integration/elite-base/packaging/aur).
They are ordinary split packages: one `pkgbase` produces the compositor, every
companion and the `better-desktop` meta package.

## Package base

| AUR repository | Source | Packages |
| --- | --- | --- |
| `niri-better` (published) | pinned commit `07e14ac3` | `niri-better`, `better-portal`, `better-companion`, `better-monitor`, `better-session`, `better-shot`, `better-wallpaper`, `better-settings`, `better-audio`, `better-sidepanels`, `better-noctalia-plugins`, `better-desktop` |
| `niri-better-git` (local recipe) | `integration/elite-base` | same split packages |

AUR only allows one package base per split package name. Publishing
`niri-better-git` alongside `niri-better` is rejected because it would overwrite
`niri-better`, `better-shot`, etc. The `-git` recipe remains in the tree for
local builds; a published variant needs `-git`-suffixed package names and
matching `provides`/`conflicts`.

The compositor packages provide/conflict with `niri`, so only one compositor
package may be installed. The tool packages are independent and do not pull the
compositor in.

## Before uploading

1. `packaging/scripts/verify.sh` must pass. It stages every split package,
   checks the file manifests, binary ownership, dependencies and completions.
2. For the pinned recipe, bump `_commit`, `pkgver` and `pkgrel` together with
   the release. The recipe comment says which commit it expects.
3. Generate `.SRCINFO` from the recipe, never by hand:

   ```sh
   cd packaging/aur/niri-better
   makepkg --printsrcinfo > .SRCINFO
   ```

4. Copy the recipe files the AUR repository needs: `PKGBUILD`, `.SRCINFO`,
   `better-desktop.install`, `better-desktop.README.md` and
   `better-settings.desktop`.

## Upload

Each AUR package base is its own git repository. Create it on the AUR web
interface once, then push:

```sh
git clone ssh://aur@aur.archlinux.org/niri-better.git
cd niri-better
# copy PKGBUILD, .SRCINFO and the helper files here
git add PKGBUILD .SRCINFO better-desktop.install \
  better-desktop.README.md better-settings.desktop
git commit -m "Update to <version>"
git push
```

Repeat for `niri-better-git`. The package names above appear automatically;
only the two package bases are separate AUR repositories.

Do not commit build outputs, `src/`, `pkg/`, or generated package tarballs.
`.SRCINFO` is required and must match the `PKGBUILD`.

## Release checklist

```sh
cd better-niri
./packaging/scripts/verify.sh
cargo test --workspace
cargo test -p niri --no-default-features
./scripts/test-all.sh
```

Then bump the pinned commit, regenerate `.SRCINFO`, push the published AUR
repository, and tag the source releases. The `-git` recipe intentionally
follows the integration branch and stays local until it gets its own split
package names.
