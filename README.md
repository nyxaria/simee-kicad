# simee-kicad

simee's fork of KiCad. This branch, `simee-ci` (the default), holds only simee's tooling. The
other branches and tags are KiCad's history, so clone with `--single-branch`.

## What it produces

GitHub releases named `cli-<kicad version>-<n>` (for example `cli-10.0.6-1`), each holding a trimmed
`kicad-cli` from that official, **unmodified** KiCad release. It contains only what `kicad-cli sch ...`
needs: the schematic module and its shared libraries.

| asset | contents |
|---|---|
| `kicad-cli-<v>-macos-arm64.tar.gz`, `-macos-x86_64.tar.gz` | `KiCad.app` with `Contents/MacOS/kicad-cli` (ad hoc signed) |
| `kicad-cli-<v>-windows-x86_64.zip` | `bin\kicad-cli.exe` and its DLLs |
| `kicad-cli-<v>-linux-x86_64.tar.gz` | `bin/kicad-cli` (a wrapper), `libexec/`, and every library but glibc in `lib/`; needs glibc 2.39+ (Ubuntu 24.04, Debian 13) |
| `kicad-<v>-source.tar.gz` | the matching KiCad source (GPL-3.0-or-later) |
| `kicad-cli-<v>-linux-x86_64-sources.tar` | the exact Debian source of every library in the Linux bundle |
| `SHA256SUMS` | checksums of everything above |

simee-core pins one release in `cmake/SimeeKicad.cmake`. simee-kicad-watcher packages each new stable KiCad
release monthly and opens the pin-bump PR.

Run `kicad-cli` with `KICAD_CONFIG_HOME`, `KICAD_DOCUMENTS_HOME` and `KICAD_CACHE_HOME` pointing at
private dirs, or its first run writes into the user's home.

## Building a release

Actions → **package** → run with a KiCad version, or `gh workflow run package.yml -f kicad_version=10.0.6`.
Locally (macOS; the Windows bundle builds anywhere with 7-Zip but only smoke-tests on Windows; the
Linux bundle builds and smoke-tests anywhere with docker):

```bash
uv run kicad-bundle --kicad-version 10.0.6 --platform macos   # -> dist/
uv run pytest
```

The download cache (`~/.cache/kicad-bundle`, or `--cache`) is pruned after every build: it keeps the
installers of the version just built and of the newest other version, and the Debian source files a
build used in the last 90 days.

How it works: download the official installer (cached in `~/.cache/kicad-bundle`), copy out the app,
walk the shared-library closure of `kicad-cli` + the eeschema kiface (`otool -L` / PE imports), drop
everything else, thin and re-sign per architecture on macOS, then export the netlist of a known
RC filter and compare KiCad's nets before archiving.

Linux has no official relocatable build, so the Linux bundle comes from the official `kicad/kicad:<v>`
Docker image (Debian, amd64 only): `docker export` its filesystem, walk the ELF `DT_NEEDED` closure of
`kicad-cli` + the kiface, and copy every library except glibc into `lib/` under the name the loader
asks for. `bin/kicad-cli` sets `LD_LIBRARY_PATH` and `KICAD_STOCK_DATA_HOME` (KiCad otherwise looks for
its data in `/usr/share/kicad`) and runs `libexec/kicad-cli`. The smoke test runs in a bare `ubuntu:24.04`
container, which proves both the glibc floor and that nothing is missing from `lib/`. The bundle is
larger than the macOS one (about 150 MB) because eeschema links wx's webview, which pulls in WebKitGTK.

## Licences

The bundles redistribute third-party libraries, several under the LGPL or GPL (cairo, glib, libgit2,
unixODBC, ...), and the combined work is GPL-3.0-or-later. So every release carries the complete
corresponding source of everything it ships rather than a written offer, which would oblige us to
supply sources on request for three years. We ship the source of every bundled library, permissive
ones included, so nothing hinges on classifying each licence correctly.

- Linux: `kicad-bundle` maps each library to the Debian package that owns it (dpkg's records in the
  image), copies that package's `copyright` file into `share/doc/<package>/`, lists them all in
  `THIRD-PARTY.txt`, and downloads each source package at the exact installed version from
  snapshot.debian.org (cached by sha1 under `~/.cache/kicad-bundle/debian-sources`). A library no
  package owns fails the build, unless it is KiCad's own (`libki*`).
- macOS (Homebrew bottles) and Windows (vcpkg): not done yet, see the open issues.

## Patching KiCad later

Nothing is patched today. If simee needs its own export command, add it on a `simee/<version>` branch
cut from the release tag and switch the package workflow from repackaging to building that branch.
Only ever add code, so the monthly rebase stays trivial.
