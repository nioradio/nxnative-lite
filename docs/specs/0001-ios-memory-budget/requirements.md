# 0001 · iOS memory budget for 8 GiB devices

Status: draft → implementing
Owner: engineering (host / nvn-mtl)
Evidence date: 2026-11-17, iPhone 17e (iPhone18,5), iOS 27.2.1, 8 GiB

## Problem

The Legend of Zelda: Ocarina of Time on an 8 GiB iPhone is ended by the system
(jetsam) after 5–100 minutes of play. Four `JetsamEvent` reports on the test
device all name `DmIOS` as the largest process:

| time | resident | reason |
|---|---|---|
| 00:53 | 1.04 GB | jetsam |
| 01:03 | 3.96 GB | `vm-pageshortage` |
| 07:49 | 3.48 GB | jetsam |
| 08:07 | 3.86 GB | `fc-thrashing` |

`docs/reference/memory.md` gives the ceiling from other runs: about 6 GB on a
12 GB iPhone with `increased-memory-limit`, 3.3 GB without it. A 8 GiB device
therefore sits near 4 GB, and the game runs at 3.5–3.9 GB. There is no room to
spare, so the app must give some back.

Two app-side consumers are ours to fix, both invisible to `phys_footprint`:

- The Metal 4 pipeline archive: 54 files, **7.6 GB** in `Documents/pso_archive`,
  mmap'd (`newArchiveWithURL`). Clean file pages, so not charged to the
  footprint — but they live in the file cache, and the kernel's own word for
  the kill is `fc-thrashing`.
- Texture views rebuilt every frame because the per-texture cache holds 4
  (`VC_WAYS = 4`); each rebuild is a new object, a new residency entry and a
  new decode.

## Requirements

**R1 · The archive must fit a budget.** The pipeline prebuild must stop adding
files once the archive reaches a byte budget, and say so in the log. Pairs it
leaves are the draws'. Default budget on iOS: half of
`os_proc_available_memory()` at start, clamped to 256 MiB … 2 GiB.
`DM_PSO_ARCHIVE_MAX=<bytes>` overrides it (0 = no budget). On the Mac, no
budget unless asked.

**R2 · The texture-view cache size must be a setting.** `DM_NVN_VC_WAYS` from
1 to 64, default 4 (today's behaviour). `DM_NVN_VC_WAYS=64` is the value
`docs/reference/memory.md` records for the 8 GiB iPhone runs that finished
races (issue #1184), with the note that it "is not on main".

**R3 · Nothing changes when the settings are unset.** A run without
`DM_NVN_VC_WAYS` and without a budget reason keeps today's memory shape and
log lines, apart from the new archive-budget line.

**R4 · The knobs are documented where the others are.** `docs/settings.md`'s
env table gains both names.

## Acceptance

1. `make` clean; the iOS app builds and signs an IPA.
2. On the device, the boot log carries `[mem] pipeline archive: X.XX GB on disk,
   budget Y.YY GB` and, when the budget stops the prebuild,
   `pipeline prebuild: stopped at the archive budget …`.
3. A 20-minute session does not reach the limit: the `[mem] memory limit …`
   line keeps ≥ 300 MB of room.
4. `DM_NVN_VC_WAYS=64` runs a title-screen route without a `[RED]` line, and
   the `village` line's `made` counter falls against the same route at 4.

## Out of scope

- Reducing the guest's working set (needs the game to touch less).
- The `residency set` growth past a pool's Finalize (fixed by `13c35e90`,
  PR #2775, already in this build).
- Offline prebake (`DM_PSO_BUNDLE`): same size, different build host.
