# 0001 · Tasks

| # | Task | File | Done when |
|---|---|---|---|
| T1 | `DM_NVN_VC_WAYS`: runtime ways in the texture-view cache | `dm/gfx/nvn/nvn_mtl.m` | builds; unset = 4; 64 accepted, 0 and 100 clamped |
| T2 | Archive byte counter and `DM_PSO_ARCHIVE_MAX` budget with the stop path | `dm/gfx/nvn/nvn_mtl.m` | builds; prebuild stops and stamps |
| T3 | Document both knobs | `docs/settings.md` | table rows added |
| T4 | Build the iOS IPA on the changed host | `scripts/build_ipa_from_nsp.py` | IPA produced, shader stage reused |
| T5 | Install and measure 20 min on the 8 GiB iPhone | device | no jetsam; `[mem] memory limit` keeps ≥ 300 MB |

Ordering: T1, T2 in one edit pass (same file, both compile or neither); T3; T4;
T5 after the user installs.

Rollback: both changes are off unless their variable is set, except the iOS
budget default — `DM_PSO_ARCHIVE_MAX=0` restores an unbounded prebuild.
