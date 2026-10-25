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

### iter-9 — first-playtest crash fixed: register the loop-twin family wholesale (2026-10-26)

Symptom: first launch, login screen reached, **crash 14 s after boot** —
`[critical] [FATAL] Unresolved branch from 0x820A4C28 to 0x820A4A2C`
(inside `sub_820A4C04` in `dfkr_recomp.4.cpp`, `bne cr6 → 0x820A4A2C`).
The trap gate's compile-time tolerance held, but one triaged family is
**proven live at runtime** and needed a real fix.

Root cause (decrypted shell `default_dec.bin`, VA = `0x82000000` + offset,
capstone): `0x820A4A2C` is the head of a **shared jump-table dispatch block**
sitting in the unowned gap between registered neighbours `sub_820A49B8` /
`sub_820A4C04`:
`addi r11,r11,-0x21; cmplwi cr6,r11,0x5C; bgt cr6,0x820A4C14; lis r12,-0x7DF6;`
`slwi r0,r11,2; addi r12,r12,0x4A50; lwzx r0,r12,r0; mtctr r0; bctr` —
93-entry label table at `0x820A4A50`, bodies `0x820A4BC4..0x820A4C14`.
Neither the PDATA scan nor bl-scan can promote it (entry is only reached by
conditional branches from twin shell loops), identical to the dfkbl
`0x820A4554` class.

Fix (both sibling precedents, applied **wholesale** instead of
one-at-a-time, payload appended to `dfkr_functions.toml`):

- 38 `[functions."0x820XXXXX"]` entries — every unique target of the 101
  triaged allowlist identities (35 plain mid-block twins + the 3 dispatcher
  heads, also registered so incoming `bc`s become resolvable tail calls).
- 3 `[[switch_tables]]` overrides for the confirmed dispatchers (futari
  pattern; labels read out of the image):

  | head (entry) | table | entries | bctr (override addr) | reg |
  | --- | --- | --- | --- | --- |
  | `0x820A0C70` | `0x820A0C98` | 21 | `0x820A0C94` | r11 |
  | `0x820A4A2C` | `0x820A4A50` | 93 | `0x820A4A4C` | r11 |
  | `0x820A5044` | `0x820A5068` | 94 | `0x820A5064` | r11 |

  (`0x821CE2E8`, also in the family, verified NOT a dispatcher — it is an
  `lbzx` byte-lookup block; plain registration suffices.)

Results: regen (`-IgnoreStamp`) + build clean (exe + 7 DLLs, 06:21
timestamps); gate re-run → **all 101 prior identities stale, 0 embedded
trap sites, 849 switch-traps informational** — the crash `bc` now compiles
as a clean tail call `sub_820A4A2C(ctx, base)`. Allowlist kept as-is
(stale entries are informational; prune on a later pass).

Known CLI noise during regen (unchanged from iter-8):
`BaseHeap::Release failed because address is not a region start` ×7 and an
exit-1 from the CLI's shutdown path despite `2 written, 172 unchanged` +
`Done` — validate/build steps confirm success independently.

Status: fixed + committed; awaiting re-playtest. If a NEW `[FATAL]
Unresolved branch` appears on a different address, add that address per the
now-proven recipe (capstone-decode context → register target; `[[switch_tables]]`
only if an `lwzx/mtctr/bctr` dispatcher head is confirmed).

### iter-10 — first clean playtest; mode→module ground truth (2026-10-26)

Re-test after iter-9: **all 7 modes entered and played, replay save and
replay load both work**. No `[error]`/`[critical]` from core, no `[FATAL]`,
no unresolved-branch events anywhere in 197 log lines.

`XexLoadImage`/`XexUnloadImage` ground truth (strict unload→load on every
switch, `load_count now 0` + `recomp=found` on every unload):

| mode | module | mount |
| --- | --- | --- |
| ver 1.5 Novice | `dat163.bin` | `game:\` |
| ver 1.51 Xbox 360 | `dat111.bin` | `ver151:\` (DLC 60C58708) |
| ver B Arrange B | `dat222.bin` | `game:\` |
| ver L Arrange A | `dat112.bin` | `game:\` |
| ver 1.5 Xbox 360 | `dat12.bin` | `game:\` |
| Black Label | `dat15.bin` → `dat164.bin` | `bl:\` (DLC 456866D4) |
| Black Label Novice | `dat15.bin` | `bl:\` |
| BL replay save/load | `dat15.bin` | `bl:\` |

Black Label detail: entering full BL mounts `dat15` (~11 s) then switches
to `dat164` (~15 s session) — `dat15` reads as the BL boot/Novice-side XEX,
`dat164` the full BL game; BL Novice and replay sessions run on `dat15`
alone (was reached by the ~15 s/per-mode test pattern; one targeted retest
pins it if ever needed). The disc-referenced `dat170.bin` and DLC
`dat162.bin`/`dat00.bin` never load — their (invisible/unowned) entries did
not surface in this pass.

Residual, non-blocking noise (`[error]`, rate-limited, game continued):

- `[gpu] Unsupported resolve vertex buffer format (vf0: type=2, size=…)` +
  `PM4_DRAW_INDX(3,4,0) Failed in backend` during the first mode boot
  (t≈06:24:25, after dat163 load).
- `[gpu] Tried to resolve to format k_1_REVERSE, which is not a
  ColorFormat` during the Black Label session (t≈06:27:41).
- `[krnl] Unimplemented XLIVEBASE message app=000000FC msg=0005000C` ×2
  during replay save/load ops — replay ops succeeded regardless.

Status: **project validated.** Pushed to GitHub after this pass.

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
