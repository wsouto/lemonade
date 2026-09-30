# Local Lemonade packaging (Arch Linux)

Links:
[Arch Package Guidelines](https://wiki.archlinux.org/title/Arch_package_guidelines) |
[AUR Submission Guidelines](https://wiki.archlinux.org/title/AUR_submission_guidelines)

Personal checkout of the **official Arch packaging repo** for Lemonade, used to
build the newest upstream release locally before it lands in `extra`.

- Packaging repo: `https://gitlab.archlinux.org/archlinux/packaging/packages/lemonade`
- Upstream: `https://github.com/lemonade-sdk/lemonade`
- Split packages: `lemonade-server` (daemon + CLI + web launcher) and
  `lemonade-desktop` (native Tauri app)

## Last build

| | |
| --- | --- |
| Version | `2026.40.0-1` |
| Upstream ref | `candidate-v2026.40.0` (commit `29b21e73`) |
| Built | 2026-09-25, Node `v26.10.0` (vite-plus), mbedTLS 4.2 |
| Artifacts | `lemonade-server-2026.40.0-1-x86_64.pkg.tar.zst`, `lemonade-desktop-2026.40.0-1-x86_64.pkg.tar.zst` |

This build:

- Installs `lemond` (daemon), `lemonade` (CLI) and `lemonade-app` (native GUI).
  The old `lemonade-server` binary name no longer exists.
- systemd service is `lemond.service`; config `etc/default/lemond`.
- Both menu entries are shipped as upstream intends: "Lemonade App"
  (`lemonade-desktop`) and "Lemonade Web App" (`lemonade-server`).

## Deviations from the upstream PKGBUILD

Two local edits, committed locally on `main`. Re-check them after any
upstream packaging update (line numbers below are for the current revision):

1. **Version / tag** (`PKGBUILD:13-15`) — builds a specific upstream ref, which
   may be a *candidate* prerelease:

   ```
   pkgver=2026.40.0
   pkgrel=1
   _tag=candidate-v${pkgver}
   ```

2. **`.version` file** (`PKGBUILD:55`, in `prepare()`) — upstream uses calendar
   versioning (`YYYY.WW.N`). `tools/version.py` only recognizes tags matching
   `^vYYYY.WW.N$`; a `candidate-v…` tag is not recognized and CMake **aborts**
   if it cannot compute a version. Writing `.version` (override priority 1)
   makes the binaries report exactly `${pkgver}`:

   ```
   printf '%s\n' "${pkgver}" > .version
   ```

The packaging otherwise matches upstream (no web-app entry removal).

## Updating to a new release

### 0. Refresh the packaging repo

The PKGBUILD / patches / package layout are maintained upstream. The local
deviations are a commit on `main`, so pull and replay it on top:

```bash
git fetch origin
git rebase origin/main     # re-resolve PKGBUILD, re-apply the local deviations
```

This checkout is local only — never push to `origin` (that is upstream's repo).
Build artifacts are ignored via the AUR-style `.gitignore` (ignore all,
whitelist package files), so `git status` stays clean after a build.

### 1. Find the ref to build

```bash
curl -s 'https://api.github.com/repos/lemonade-sdk/lemonade/releases?per_page=10' \
  | jq -r '.[] | "\(.tag_name)\tprerelease=\(.prerelease)"'
```

- Stable tag `v2026.41.0` → `pkgver=2026.41.0`, `_tag=v${pkgver}`
- Candidate prerelease `candidate-v2026.41.0` → `pkgver=2026.41.0`,
  `_tag=candidate-v${pkgver}`

> Candidates often carry fixes that matter on Arch (e.g. mbedTLS 4 support).
> If a stable tag predates an Arch-relevant fix, prefer the candidate.
> Check `pacman -Si mbedtls` (Arch ships v4) against the release notes.

To pin a commit instead: drop `_tag`, use
`git+${url}.git#commit=<sha>` in `source`.

### 2. Edit the PKGBUILD

- `pkgver=<new>`, `pkgrel=1`
- `_tag=v${pkgver}` (stable) or `_tag=candidate-v${pkgver}` (candidate)
- Leave the `.version` line as-is; it follows `pkgver` automatically.
- Also diff against the upstream `PKGBUILD` (from step 0) in case
  `makedepends`, patches or the `package_*()` functions changed.

### 3. Regenerate checksums and metadata

```bash
updpkgsums                              # regenerates b2sums (patch sums stay)
makepkg --printsrcinfo > .SRCINFO       # refresh tracked metadata
```

### 4. Re-check the patches

The two patches are applied in `prepare()`. `lemonade-sysusers.patch` is stable.
`lemonade-web-app.patch` touches `CMakeLists.txt` and may need a rebase; test it
against the new tag before the long build:

```bash
REPO=$(pwd)                       # run from this packaging directory
rm -rf /tmp/lemcheck && mkdir -p /tmp/lemcheck
git --git-dir=./lemonade archive --format=tar <tag> | tar -x -C /tmp/lemcheck
cd /tmp/lemcheck
patch -Np1 --dry-run -i "$REPO/lemonade-sysusers.patch"
patch -Np1 --dry-run -i "$REPO/lemonade-web-app.patch"
```

(`./lemonade` is the git source cache left by `updpkgsums`/`makepkg`.)

A hunk "succeeded with offset/fuzz" is fine; a failed hunk means updating the
patch's context to match. The patch exists so the two split packages do not both
install `/usr/share/pixmaps/lemonade-app.svg` / `Icon=lemonade-app` (Arch
packaging commit `35f8b64`). **Keep it** — it renames the server package's
pixmap/icon so both split packages can be installed side by side.

### 5. Preflight dependencies and Node

```bash
node --version     # must be v26.10.0 (vite-plus), NOT /usr/bin/node (24)

awk '/^\tmakedepends = /{print $3}' .SRCINFO | xargs pacman -T \
  || echo "install the missing packages first"
```

Why this matters: `makepkg` sources `/etc/profile` **only** when it must install
missing deps (`handle_deps()`), and that drops the vite-plus path so the build
silently switches to `/usr/bin/node` (Node 24). So install anything missing in a
separate step, reopen the shell, then build. Prefer plain `makepkg` over
`makepkg -s`.

### 6. Build, install, verify

Run from a shell where `node --version` is `v26.10.0`:

```bash
makepkg                                   # both split packages
sudo pacman -U lemonade-server-<ver>-1-x86_64.pkg.tar.zst \
               lemonade-desktop-<ver>-1-x86_64.pkg.tar.zst

lemond --version && lemonade --version    # both → <ver>
sudo systemctl daemon-reload
sudo systemctl enable --now lemond
systemctl status lemond
```

`lemonade-debug-*.pkg.tar.zst` is produced too; it is not needed.

### Quick re-package after editing only `package_*()`

No rebuild required:

```bash
makepkg -R -f --noconfirm
```

## Web-app menu entry

Two launchers exist, one per split package:

| Menu entry | Package | Exec |
| --- | --- | --- |
| "Lemonade App" (native Tauri) | `lemonade-desktop` | `lemonade-app` |
| "Lemonade Web App" (browser) | `lemonade-server` | `lemonade-web-app` |

The web launcher requires `jq` to discover the server port before browser
selection. It prefers available Chromium-based browsers with `--app`; if none
is available (or Chromium preference is disabled), it uses `xdg-open`, provided
by the optional `xdg-utils` package.

Both entries are shipped by default. If you ever want only one:

### Keep only the native app (drop the web-app entry)

Add to `package_lemonade-server()` (after the `cmake --install` line):

```bash
rm -f "${pkgdir}/usr/share/applications/lemonade-web-app.desktop"
```

then `makepkg -R -f --noconfirm` and reinstall `lemonade-server`.

### Keep only the web app

```bash
sudo pacman -Rns lemonade-desktop
```

### Hide the web-app entry but keep it functional

Ship it with `NoDisplay=true` instead of deleting it:

```bash
printf '\nNoDisplay=true\n' >> "${pkgdir}/usr/share/applications/lemonade-web-app.desktop"
```

## Gotchas

- Candidate tags (`candidate-v…`) are not valid `pkgver` values and are not
  recognized by upstream's `tools/version.py`; the `_tag` + `.version` pair
  handles both.
- `makepkg` (7.1) has no per-split-package build; `build()` always compiles the
  Tauri app.
- First build downloads Rust crates and may fetch some CMake `FetchContent`
  deps (brotli); allow network.
- If `extra`'s `mbedtls` is v4 and the target release predates the mbedTLS 4
  fix, the build fails — pick a newer ref.
- `backup=(etc/default/lemond)` is what `pacman` preserves on upgrade. Config
  is `/etc/default/lemond` (older docs may say `/etc/lemonade/…`).
