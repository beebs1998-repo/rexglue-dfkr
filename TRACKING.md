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
  `456866D4E784B661072F7B2D51C63D94E088644754`; `spa.bin` in the same
  package is an XDBF asset container, not a module. `dat111.bin` is in
  package `60C587083B6BAC92A1F662AF72A11BD497F4838554`.
- NOT available locally (cannot recomp): `dat170.bin` (disc path referenced
  by the shell), `dat162.bin`, `dat00.bin` (DLC entries without an owned
  package). Standing: the affected modes are explicitly missing until those
  images surface; first playtest to confirm which menu entries fail.
- Codegen manifest set up for entrypoint + 7 modules; per-target codegen
  (shared base `0x88000000`) with committed hand-merged glue, per the
  siblings.

Status: scaffold + staging done; toolchain builds, first codegen and first
launch pending.
