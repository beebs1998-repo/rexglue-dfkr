# DFKR Recomp

Recompilation of the Cave Xbox 360 shooter *DoDonPachi DaiFukkatsu
Resurrection* (XBLA title `545A07D2`) using
[ReXGlue](https://github.com/rexglue/rexglue-sdk).

This is the fourth title in the family of ReXGlue recomp projects, sharing its
methodology (and its SDK fork) with the sibling ports:

- [rexglue-futari](https://github.com/beebs1998-repo/rexglue-futari) —
  *Mushihimesama Futari* (first project)
- [rexglue-dfkbl](https://github.com/beebs1998-repo/rexglue-dfkbl) —
  *DoDonPachi DaiFukkatsu Black Label* (second project)
- [rexglue-mushi](https://github.com/beebs1998-repo/rexglue-mushi) —
  *Mushihimesama* (third project)

The project builds a native `dfkr.exe` (plus seven game-mode module DLLs, three
delivered as DLC) from statically recompiled PowerPC code. **The recompiled
code and a patched ReXGlue SDK are included in this repository**, so the
project builds on a clean machine without the game. Only *running* it needs
your own copy of the game data.

For the engineering history (what was fixed and why), see
[TRACKING.md](TRACKING.md). Agent-/developer-facing notes live in
[AGENTS.md](AGENTS.md).

## Requirements

**To build** — no game files needed:

- CMake 3.25 or newer
- Ninja
- LLVM/Clang with C++23 support
- Windows: Visual Studio C++ build tools with a Windows SDK
- Linux: `libgtk-3-dev libx11-xcb-dev libvulkan-dev libasound2-dev libpulse-dev libpipewire-0.3-dev`
  (Debian/Ubuntu names); the `linux-*` presets expect `clang-20`/`clang++-20`

**To run** — your own copy of the DoDonPachi Resurrection XBLA disc, extracted
into `game_data/`:

```
game_data/
├─ default.xex          # entrypoint (shell)
├─ dat12.bin            # disc game-mode module
├─ dat112.bin           # disc game-mode module
├─ dat163.bin           # disc game-mode module
├─ dat222.bin           # disc game-mode module
├─ dat111/164/15.bin    # LOCAL staging copies of the DLC modules (see below)
├─ res/DISKDATA/...     # disc data
├─ AvatarAssetPack
└─ nxeart
```

Four disc modules are swapped mutually exclusively on mode change
(unload→load); the DLC ones share the same guest base and are mounted through
their own content packages.

### DLC

The retail edition's extra game modes ship as 00000002 (marketplace content)
packages. Three of their binaries are recompiled here:

| Content id | Module | Notes |
| --- | --- | --- |
| `60C587083B6BAC92A1F662AF72A11BD497F4838554` | `dat111.bin` | package display name: **"ver 1.51"** |
| `456866D4E784B661072F7B2D51C63D94E088644754` | `dat164.bin` | package display name: **"DoDonPachi Resurrection Black Label"** |
| `456866D4E784B661072F7B2D51C63D94E088644754` | `dat15.bin` | same package (also carries `spa.bin`, XDBF asset data) |

Availability comes entirely from *staging* — no config cvar:

1. Copy the package so it matches the SDK `ContentManager` layout:
   `user_data/0000000000000000/545A07D2/00000002/<content-id>/` plus
   `Headers/00000002/<content-id>.header` (xuid forced to 0).
2. Append a 4-byte `0xFFFFFFFF` license tail to the staged `.header` (the
   `LoadPackageLicenseMask`/`WriteContentHeaderFile` tail convention; see
   futari TRACKING.md Iteration 20).
3. `license_mask` must not be set anywhere - the cvar is dead and removed.

For local codegen, the DLC modules also need staging copies at
`game_data/dat111.bin`, `dat164.bin`, `dat15.bin` (see the manifest comment;
described in this repo, not committed).

## Layout

| Path | What |
| --- | --- |
| `src/` | Host application (`main.cpp`, `dfkr_app.h`) |
| `generated/default/` | Recompiled entrypoint + `dll_targets.cmake` / `module_registry.cpp` |
| `generated/dat12\|dat112\|dat163\|dat222\|dat111\|dat164\|dat15/` | Recompiled game-mode modules |
| `generated/rexglue.cmake` | SDK/CMake glue (codegen auto-generation disabled — see below) |
| `dfkr_manifest.toml` | Codegen manifest (entrypoint + 7 modules) |
| `dfkr*_functions.toml` | Extra dynamic entrypoints registered per target |
| `dfkr.toml` | Settings template — the live copy is staged next to the exe; editable via the F4 overlay |
| `thirdparty/rexglue-sdk/` | Patched ReXGlue SDK (git submodule, pinned) |
| `scripts/codegen_all.ps1` | Regenerate all recompiled code (needs the game) |
| `scripts/build_module_hints.ps1` | Static function-entry hint derivation (needs the game, once per machine) |
| `scripts/validate_codegen.ps1` | Embedded-trap gate — runs after codegen and in CI |
| `scripts/stage_config.cmake` | Copies `dfkr.toml` next to the exe (if missing) — see "Settings" |
| `TRACKING.md` | Iteration log (bugs found + fixes) |

## Building

Clone with the submodules:

```
git clone --recurse-submodules https://github.com/beebs1998-repo/rexglue-dfkr.git
cd rexglue-dfkr
```

(Already cloned without them? Run `git submodule update --init --recursive`.)

The SDK submodule itself pulls ~20 upstream dependencies on first init, so
this step can take a few minutes.

### Windows

```
cmake --preset win-amd64-release
cmake --build --preset win-amd64-release --parallel
```

### Linux

```
cmake --preset linux-amd64-release
cmake --build --preset linux-amd64-release --parallel
```

The build places the executable and all module/runtime DLLs in
`out/build/<preset>/`. The settings template (`dfkr.toml`) is staged next to
the executable on first build (if not already there).

## First launch

1. Put your extracted game data in `game_data/` (see Requirements).
2. Either run `launch.bat` (Windows), or start the executable directly:

```
out\build\win-amd64-release\dfkr.exe --game_data_root="game_data" --user_data_root="user_data"
```

The game writes logs to `logs/` and saves to `user_data/`. Xenia Canary
exported saves/replays (the `content/` tree) can be copied straight into
`user_data/` per the title-id/device directory layout - the SDK accepts both
raw `XCONTENT_AGGREGATE_DATA` dumps and full STFS/CON package headers.

## Settings

The game reads and writes `dfkr.toml` **next to the executable**
(`out\build\win-amd64-release\dfkr.toml` by default). The copy in this
repository is a *template*: the build (POST_BUILD) and `launch.bat` stage it
next to the exe **only if the live file does not already exist**, so the
curated defaults are in effect from the very first launch. A copy-if-missing
staging rule means:

- your F4 edits and manual tweaks are never clobbered by a rebuild;
- editing the repo template after first stage does NOT propagate - delete
  the live copy (or re-edit via F4) to re-sync.

Saving is **diff-only**: only cvars whose current value differs from the
compiled-in SDK default are written. A cvar sitting at its default therefore
does not appear in the file, and a default-equal line is dropped by the next
save - that is expected, not a desync between the menu and the file.

Current notable options (bundled SDK = pristine mainline plus the fixes
documented in AGENTS.md):

| Option | Default | Effect |
| --- | --- | --- |
| `d3d12_submit_on_primary_buffer_end` | `false` | D3D12 submission at primary buffer end; `false` is the validated slowdown fix for this title family (diff-only save usually omits it) |
| `vsync` | `true` | Present sync; keep on for correct 60 Hz guest pacing |
| `gpu_allow_invalid_fetch_constants` | `true` (template) | Accept invalid Xenos fetch constants instead of failing the draw |
| `mnk_mode` | `false` | Keyboard/mouse controller emulation (enabled in the template) |
| `log_file` etc. | sequential files | Template pins a fixed rotating `logs/dfkr.log` (see the template header) |

## Regenerating the recompiled code (optional)

Only needed if you change the manifest/function lists, or bump the SDK. It
requires the game files **and** the `rexglue` CLI (built with the SDK).

```
.\scripts\codegen_all.ps1
```

The script ends with the embedded-trap gate (`scripts/validate_codegen.ps1`):
a codegen turn that emits a new recompiler-trap site fails there instead of
as a runtime crash (see `scripts/trap_allowlist.txt` for the triaged
baseline). A one-time per-machine build of the XenonRecomp submodule
(`pwsh .\scripts\bootstrap_xenon.ps1`) is also needed only for
`scripts/build_module_hints.ps1` - the static missing-function derivation
that seeds the `dfkr*_hints.toml` files.

Two facts carried over from the sibling projects:

- The seven modules share guest base `0x88000000`, so codegen refuses to
  process them in a single run; the script runs one target per invocation.
- `codegen` rewrites `generated/default/dll_targets.cmake`,
  `generated/default/module_registry.cpp` and `generated/rexglue.cmake` for
  the *last* target only. The committed copies are hand-merged (all seven
  modules) and auto-codegen is disabled in `rexglue.cmake`. The script
  restores them (`git checkout -- ...`) after running; don't let these get
  clobbered.

## Known issues

See [TRACKING.md](TRACKING.md) for the current open items and the
playtest-driven iteration log. Day-one status:

- the shell references `game:\dat170.bin` and DLC entries `dat162.bin` /
  `dat00.bin` that are NOT present in the local `game_data/`/user packages -
  the affected modes (exactly which, to be confirmed at first playtest) are
  unavailable until those images surface.

## Credits & legal

- Runtime/toolkit: [ReXGlue](https://github.com/rexglue/rexglue-sdk); this
  repo pins a lightly patched fork (`thirdparty/rexglue-sdk`, shared by the
  sibling ports) with the fixes documented in AGENTS.md.
- **No game data or DLC packages are included.** You must own DoDonPachi
  Resurrection (and the DLC packages if you want the extra modes) and extract
  your own `game_data/`. The recompiled code in `generated/` is derived from
  that game and is provided for interoperability/archival research; use only
  with legally owned data.
