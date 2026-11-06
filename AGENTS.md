# AGENTS.md

## What this repo is

Automated personal Arch Linux binary repository. GitHub Actions builds packages and publishes them to a fixed GitHub Release tag `repository` (pacman DB name `arcox`). Everything is driven by the single file `packages.txt` at the repo root.

One directory per package, each containing a `PKGBUILD`.

## Controlling which packages exist — `packages.txt`

- Plain lines (e.g. `opennow`, `halley`) = **AUR/CachyOS packages**, synced automatically.
- Prefixed lines (e.g. `local:meiro-bin`) = **local packages**, maintained manually in this repo.
- Comments start with `#`; `local:` entries are stripped of the prefix when matching directories.
- **Sync workflow (`sync-aur.yml`) runs hourly and rewrites AUR package directories from upstream** (CachyOS-PKGBUILDS, falling back to aur.archlinux.org). Anything hand-edited inside an AUR package's directory is silently overwritten. Only edit local packages.
- The workflow also **deletes any directory not listed in `packages.txt`** (e.g. leftover dirs). Don't add package dirs without adding them to `packages.txt`.
- `local:` entries are skipped by sync, so they are safe to edit.

## Two kinds of local packages

Both are `local:` entries but differ in how they're built:

1. **Template packages** — PKGBUILD contains `@…@` placeholders (e.g. `ghostline-bin` has `pkgver=@PKGVER@`, `source=("@SOURCE@")`).
   - The committed file is a template, *not* buildable as-is.
   - `build-release.yml` renders it at build time: it clones the repo named in `url=`, reads its latest release tag, and runs the upstream `build/linux/aur/render.sh` to fill the placeholders and download the source tarball.
   - **Keep the placeholders in the committed file.** Do not expand them to literal values.
2. **Normal local packages** — full PKGBUILD, built directly (e.g. `meiro-bin`, `cressida-git`, `antseed-desktop`, `xdg-desktop-portal-halley`, `radmin-vpn-linux`).

## Version quirks

- Use `pkgver()` for dynamic versioning. When you define `pkgver()`, **keep a `pkgver=0` placeholder line** — makepkg's lint fails with "pkgver is not allowed to be empty" if the variable is unset, and `pkgver()` only runs during a real build (not `--printsrcinfo`, which will still show `0`).
- For dynamic sources that can't be checksummed, use `sha256sums=('SKIP' ...)`.
- Local packages must all carry maintainer `hphuc25 <https://github.com/hphuc25>` (AUR packages inherit their upstream maintainer and should be left alone).

## Build/publish pipeline (`build-release.yml`)

- Builds in container `cachyos/cachyos-v3` → binaries target **x86-64-v3** CPUs.
- The `prepare` job scans every dir with a `PKGBUILD` (except `.github`, `cache`, `output`) into a build matrix.
- Caches already-published packages from the release, falls back to the previous release's binaries if a build fails.
- Publish job runs `repo-add` (DB name `arcox`) on fresh assets, cleans the release's old assets, and uploads to the `repository` tag. Only triggered when `has_new_changes=true`.

## Environment

- `.gitignore` excludes `src/`, `pkg/`, `*.pkg.tar.*` (build artifacts), and `NOTE.md`.
- `.github/` contains only `build-release.yml` and `sync-aur.yml`. (README mentions a `verify-change.yml`, but that workflow does not exist in this repo — trust the actual `.github/workflows/`.)
- No tests, linters, or build scripts exist for local development; the only executable pipeline is the GitHub Actions workflows.
