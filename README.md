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
| `kicad-<v>-source.tar.gz` | the matching KiCad source (GPL-3.0-or-later) |
| `SHA256SUMS` | checksums of everything above |

simee-core pins one release in `cmake/SimeeKicad.cmake`. simee-kicad-watcher packages each new stable KiCad
release monthly and opens the pin-bump PR. Linux isn't packaged yet.

Run `kicad-cli` with `KICAD_CONFIG_HOME`, `KICAD_DOCUMENTS_HOME` and `KICAD_CACHE_HOME` pointing at
private dirs, or its first run writes into the user's home.

## Building a release

Actions → **package** → run with a KiCad version, or `gh workflow run package.yml -f kicad_version=10.0.6`.
Locally (macOS; the Windows bundle builds anywhere with 7-Zip but only smoke-tests on Windows):

```bash
uv run kicad-bundle --kicad-version 10.0.6 --platform macos   # -> dist/
uv run pytest
```

How it works: download the official installer (cached in `~/.cache/kicad-bundle`), copy out the app,
walk the shared-library closure of `kicad-cli` + the eeschema kiface (`otool -L` / PE imports), drop
everything else, thin and re-sign per architecture on macOS, then export the netlist of a known
RC filter and compare KiCad's nets before archiving.

## Patching KiCad later

Nothing is patched today. If simee needs its own export command, add it on a `simee/<version>` branch
cut from the release tag and switch the package workflow from repackaging to building that branch.
Only ever add code, so the monthly rebase stays trivial.
