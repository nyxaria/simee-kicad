# simee-kicad

simee's fork of KiCad. This branch, `simee-ci` (the default), holds only simee's tooling. The
other branches and tags are KiCad's history, so clone with `--single-branch`.

## What it produces

GitHub releases named `cli-<kicad version>-<n>` (for example `cli-10.0.6-1`), each holding a trimmed
`kicad-cli` from that KiCad release. It contains only what `kicad-cli sch ...` needs: the schematic
module and its shared libraries. Up to `cli-10.0.6-3` they are the official, unmodified binaries; a
release built with `--simee-ref simee/<version>` has KiCad's own files built from that branch (see
"Patching KiCad") on the official release's third-party libraries, and its source asset is the
branch's.

| asset | contents |
|---|---|
| `kicad-cli-<v>-macos-arm64.tar.gz`, `-macos-x86_64.tar.gz` | `KiCad.app` with `Contents/MacOS/kicad-cli` (ad hoc signed) |
| `kicad-cli-<v>-windows-x86_64.zip` | `bin\kicad-cli.exe` and its DLLs; needs Windows 10 or later |
| `kicad-cli-<v>-linux-x86_64.tar.gz` | `bin/kicad-cli` (a wrapper), `libexec/`, and every library but glibc in `lib/`; needs glibc 2.39+ (Ubuntu 24.04, Debian 13) |
| `kicad-<v>-source.tar.gz` | the matching KiCad source (GPL-3.0-or-later): the official tag's, or the `simee/<version>` branch's |
| `kicad-cli-<v>-linux-x86_64-sources.tar` | the exact Debian source of every library in the Linux bundle |
| `kicad-cli-<v>-macos-sources.tar` | the source of every third-party library in the macOS bundles, with Homebrew's formulae and patches |
| `kicad-cli-<v>-windows-x86_64-sources.tar` | the upstream sources of every vcpkg port in the Windows bundle, with the ports (portfiles, patches) |
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

The macOS build reads homebrew-core's history from GitHub's API: set `GITHUB_TOKEN` (for example
`GITHUB_TOKEN=$(gh auth token)`), or it runs into the 60 requests an hour allowed without one.

The download cache (`~/.cache/kicad-bundle`, or `--cache`) is pruned after every build: it keeps the
installers of the version just built and of the newest other version, and the source files a
build used in the last 90 days (macOS: the third-party sources, and the UUIDs, formula and SBOM of
each Homebrew bottle tried; only the bottles a `--simee-ref` build poured are kept). The treeless clones of the vcpkg
registries (`vcpkg-registries/`, a few MB) aren't pruned.

How it works: download the official installer (cached in `~/.cache/kicad-bundle`), copy out the app,
walk the shared-library closure of `kicad-cli` + the eeschema kiface (`otool -L` / PE imports), drop
everything else (on Windows also the app-local Universal CRT, `api-ms-win-*.dll` and `ucrtbase.dll`,
which Windows 10 and later never load), thin and re-sign per architecture on macOS, then export the netlist of a known
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
- macOS: KiCad's builder (kicad-mac-builder) takes most libraries from Homebrew bottles, relinked and
  re-signed, which keeps each library's Mach-O UUID. `kicad-bundle` maps each library to its formula
  (`homebrew.FORMULAE`), walks the formula's homebrew-core history back from the KiCad release until
  the bottle for the macOS the library was built for holds a file with the same UUID, and takes the
  source archive that bottle's SBOM names (or, in a bottle older than Homebrew's SBOMs, its formula
  names), the patches its formula applies, and the formula itself.
  wxWidgets (KiCad's fork), ngspice and Python are built by kicad-mac-builder: their pins come from its
  release branch (`10.0` for 10.0.x) as it was when KiCad published the release, a pinned branch
  resolves to its head at that moment, and the version string in the binary must match the pin. Each
  bundle's `THIRD-PARTY.txt` lists file -> component -> where it came from, with the licence files
  of each component's source in `KiCad.app/Contents/Resources/Licenses/<component>/`, and
  `kicad-cli-<v>-macos-sources.tar` holds the sources of both architectures. A library that is none
  of these, nor KiCad's own (`libki*`), fails the build.
- Windows: KiCad builds its DLLs with vcpkg in manifest mode, from `vcpkg.json` (version overrides)
  and `vcpkg-configuration.json` (registries and baselines: microsoft/vcpkg, plus KiCad's own
  kicad-vcpkg-registry for python3, wxwidgets-33, ...) in its source at the release tag. Each DLL
  names the port it was built in (`C:\vcpkg\buildtrees\<port>\...` in its PDB path or `__FILE__`
  strings). `kicad-bundle` resolves each port's version (override, else the registry's baseline),
  reads the port folder from the registry by its git tree (`vcpkg.py`, a treeless shallow clone),
  and evaluates just enough of the portfile to find its downloads (`portfile.py`: URLs, SHA512s,
  patches; downloads behind an `if()` are included anyway, so the sources are a superset). Each
  download must match its SHA512. Checks that this is what KiCad built: a DLL that names its vcpkg
  source folder (`<ref>-<hash>.clean`; the hash covers the archive and its patches) must match a
  download of the port, and otherwise its FileVersion or ProductVersion must start with the port's
  version. `THIRD-PARTY.txt` lists file -> port -> registry and tree, the licence files of each
  port's source go to `share/doc/<port>/`, and `kicad-cli-<v>-windows-x86_64-sources.tar` holds each
  port's downloads and the port folder itself. A DLL that names no port, nor is KiCad's own
  (`kicad-cli.exe`, `_*.dll`, `ki*.dll`) or Microsoft's C++ runtime (`vcruntime140*`, `msvcp140*`,
  ..., checked by its version resource), fails the build. The C++ runtime is shipped as KiCad ships
  it, as Visual Studio 2022 Distributable Code, and the notice says so. Decided in #9: simee keeps
  shipping it rather than requiring the VC++ Redistributable, on George's own Visual Studio licence
  (redistribution is limited to licensed Visual Studio users). Those terms also require whoever
  ships simee to external users to bind them to terms protecting Microsoft's code at least as much.

## Patching KiCad

simee's changes to KiCad live on `simee/<version>` branches cut from the release tag (`simee/10.0.6`),
one commit per change so the monthly rebase onto the next release stays trivial. Prefer backporting an
upstream commit over writing our own, and drop it once the release that has it is tracked.

`simee/10.0.6` holds:
- `kicad-cli sch import`, backported from KiCad master (upstream 473474c51a3a, in KiCad 11): imports
  Altium, Eagle, CADSTAR, EasyEDA (Std and Pro), LTspice and PADS schematics and saves them as
  `.kicad_sch`, which `sch export netlist` then reads. The output folder must exist.
- an EasyEDA Std import fix: circle net flags (`part_netLabel_Bar`) now connect (not fixed upstream).

To build them into a bundle: `uv run kicad-bundle --kicad-version 10.0.6 --platform linux --simee-ref
simee/10.0.6` (the package workflow's `simee_ref` input does the same; it resolves the branch to one
commit for every platform). The Linux packager builds `kicad-cli`, the eeschema kiface and `libki*` from
the branch on the official `kicad/kicad` image itself, with Debian's archive as it was when the image was
made (snapshot.debian.org, from the image's dpkg status time) and every installed package held, then
checks every Debian source it built against has the version the image ships, replaces KiCad's own files
in the repackaged bundle and smoke-tests `sch import` on the fixtures too. It also writes
`kicad-<v>-source.tar.gz` (GitHub's tarball of that commit). The build runs under amd64 emulation on an
arm64 Mac (slow). Windows (#13) refuses `--simee-ref` until it can build it, so no release mixes
patched and unpatched platforms.

macOS (`kicad_bundle/macos_build.py`): the bundle is the official DMG with KiCad's own files rebuilt
from the branch, one architecture at a time (x86_64 cross-built on arm64). They're built against
exactly what the DMG ships, so nothing else changes and its third-party notices and sources still hold:
- the Homebrew bottles its libraries came from (the ones `THIRD-PARTY.txt` lists, found by Mach-O UUID),
  poured into a private prefix (`brew_prefix.py`: each keg relocated as `brew` would), plus
  opencascade's (matched the same way from the full DMG) and glm's at the release date, which KiCad's
  CMake needs too. Nothing else is searched: no other Homebrew;
- kicad-mac-builder's wxWidgets fork, configured and made with its `wx.cmake` at the pinned commit;
- the DMG's own Python.framework (with its wxPython) and ngspice, and the pinned ngspice's headers;
- kicad-mac-builder's CMake options for KiCad (`DEFAULT_INSTALL_PATH`, `KICAD_SCRIPTING_WXPYTHON`, the
  DMG's deployment target), minus translations and QA tests, which don't change the binaries.

Each built file then gets the install name, dependencies and rpaths of the official file it replaces,
and every symbol it imports from a bundled library must be exported by one, or the build fails. The smoke
test also imports every fixture in `kicad_bundle/smoke/import/` with `sch import`. Needs Xcode, CMake,
ninja and swig (`brew install swig ninja`); a local build on an M-series Mac takes about an hour, and
the `macos-14` runner several, so mind the Actions minutes and prefer building locally:

```bash
GITHUB_TOKEN=$(gh auth token) uv run kicad-bundle --kicad-version 10.0.6 --platform macos --simee-ref simee/10.0.6
```

To try a change on macOS: `dev/build-macos-homebrew.sh <simee/<version> checkout> <build dir>` builds
`kicad-cli` and the eeschema kiface against Homebrew (a dev build, not a release one), then
`KICAD_CLI=<build dir>/kicad/KiCad.app/Contents/MacOS/kicad-cli uv run pytest` also runs the import
tests, which skip without `KICAD_CLI`. They import each real circuit in `kicad_bundle/smoke/import/`
(Adafruit BME280 and SparkFun logic level converter in Eagle, Digispark ATtiny85 in Altium, Easy-SDR
coax power supply in EasyEDA), export the netlist and compare it with the source tool's: for Eagle,
read straight from the Eagle XML (`tests/eagle_nets.py`); for the others, checked by hand against the
project's own schematic export, as each `fixture.json` says. Each fixture keeps its source's licence.
