# TRACKING.md — dfkr (DoDonPachi DaiFukkatsu Resurrection, XBLA 545A07D2)

Iteration log. Format inherited from the sibling ports: each change to
runtime/GPU/module behaviour gets an entry (symptom → root cause → fix →
status). Read before changing runtime/GPU behaviour.

History and conventions live in the sibling repos:

- `rexglue-futari` — first project; jump-table / crash-diagnostics playbook.
- `rexglue-dfkbl` — second project; XENIA_CANARY_SLOWDOWN_FINDINGS
  (the D3D12 submission-style slowdown fixes that the shared SDK fork pins).
- `rexglue-mushi` — third project; config-file staging convention, DLC
  staging conventions, static hint derivation, embedded-trap gate.

## Iterations

### iter-8 — static hint pass + trap-gate triage (2026-10-24)

- `scripts/build_module_hints.ps1` full pass over all 8 targets: modules
  **21 real hints each** (dat12/dat112/dat163/dat222/dat111/dat164/dat15 —
  shrink by stubs needs a `-Quick`-based map as the sisters; shell **344 real
  hints** from a 4,646-candidate diff against an 8,812-entry registered set,
  ~93% of candidates data/gap-fill misidentifications, dropped by the stub
  filter). All 8 `dfkr*_hints.toml` wired into the manifest include lists
  FIRST (prior to `dfkr_functions.toml`), `dfkr_hints.toml` cleaned of the
  8 bootstrap seeds (they live in `dfkr_functions.toml`, matching dfkbl).
- One state artifact: the shell leg printed `unresolved-b = 56` (56 `b`
  targets neither XenonRecomp nor ReXGlue treat as function starts —
  the Cave-shell loop-twin cluster, see below) and a subsequent stamp-less
  codegen_all run reported `codegen failed for target 'dat12' (exit 1)`:
  root cause confirmed as the CLI's up-to-date check failing on the
  transient tree left by the hints pass; `--ignore-stamp` codegen of all
  7 targets then regenerated the current tree (exit 0). The committed
  code + build stay validated; stamped runs should recover.
- **Embedded-trap gate after the hint regen: 130 site occurrences / 101
  unique identities, ALL in `default`, every one a cr6-twinned
  unresolved_call / unresolved_branch site** whose target is one of the
  ~56 shell b addresses (cluster families 0x820Axxx, 0x820Dxxxx,
  0x821C9678-0x821CE3EC, 0x8235F088-0x8235F334). Triaged into
  `scripts/trap_allowlist.txt` via `-WriteAllowList` — same
  block-twin/jump-table residue class as mushi TRACKING.md Iteration 4
  (mushi's clusters share the 0x821CExx / 0x8235F-shape since these are
  common Cave-shell loop twins) and dfkbl's 145-site baseline.
  **Caveat: NOT playtest-verified yet** — dfkbl cut 20 sites later via the
  SDK branch-classification fix + a live mid-block registration. First
  playtest expected to either confirm or shrink this baseline.
- Gate now: validation OK (130 allowed, 846 switch-traps informational).
- Incremental rebuild succeeds: exe + all 7 module DLLs built against the
  hinted tree. Not yet playtested.
- Status: hints + triage committed; first playtest is the next session's
  driver (mode entries, uncertain `dat170`/`dat162`/`dat00` entrypoints).

## Known-unknowns status

- Package display names drive the mode mapping so far ("ver 1.51" →
  dat111; "DoDonPachi Resurrection Black Label" → dat164/dat15); the
  `XexLoadImage` log provides ground truth at first playtest.

### iter-0 — project bootstrap (2026-10-24)

What: repo + scaffold created from the mushi methodology; SDK fork pinned at
`beebs1998-repo/rexglue-sdk` @ `348c804`.

Findings recorded during bootstrap (static, via 360tools):

- Entry XEX (545A07D2): uncompressed+encrypted, load address `0x82000000`,
  entry `0x820E9180`, image size `0x510000`; build date 2011-09-01.
- Decrypted shell scanned for module paths (360tools `extract_pe.py` +
  strings): disc loads referenced as `game:\dat12.bin`, `game:\dat170.bin`,
  `game:\dat163.bin`, `game:\dat222.bin`, `game:\dat112.bin`; DLC table
  (next to the "Verifying Downloadable Content" string, `"%s:\%s"` mount
  format): `dat162.bin`, `dat00.bin`, `dat111.bin`, `dat164.bin`,
  `dat15.bin`.
- `dat15.bin`/`dat164.bin` are XEX2 images inside DLC package
  `456866D4E784B661072F7B2D51C63D94E088644754` (display name **"DoDonPachi
  Resurrection Black Label"**), `dat111.bin` is in package
  `60C587083B6BAC92A1F662AF72A11BD497F4838554` (display name **"ver 1.51"**).
  `spa.bin` in the 456866D4 package is an XDBF asset container, not a module.
  Package display names are the only user-visible ground truth for the
  mode→module mapping so far; `XexLoadImage` log lines confirm it at first
  playtest.
- Staged `user_data\0000000000000000\545A07D2\Headers\00000002\*.header` are
  full 0xA000 STFS volume page extracts (Xenia Canary), mostly zeroed except
  magic/title-id/license/display-name/content-id (`FFFFFFFF` license flag at
  page offset 0x148). This is what the fork's `ReadContentHeaderFile` fix
  (`9019af7`) was written to accept; no license-tail surgery planned —
  revisit only if DLC licenses fail at first playtest.
- NOT available locally (cannot recomp): `dat170.bin` (disc path referenced
  by the shell), `dat162.bin`, `dat00.bin` (DLC entries without an owned
  package). Standing: the affected modes are explicitly missing until those
  images surface; first playtest to confirm which menu entries fail.
- Codegen manifest set up for entrypoint + 7 modules; per-target codegen
  (shared base `0x88000000`) with committed hand-merged glue, per the
  siblings.

Status: scaffold + staging done; toolchain builds, first codegen and first
launch pending.
