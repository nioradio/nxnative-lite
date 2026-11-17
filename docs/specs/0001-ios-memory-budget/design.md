# 0001 · Design

Two code changes, both in `dm/gfx/nvn/nvn_mtl.m`. No ABI or serialisation change:
existing archives stay readable, and a run with neither setting behaves as today.

## D1 · Archive budget (R1)

### Where the bytes are counted

`arch_write_file()` is the only place a `.mtl4a` reaches the disk (line ~9930).
It already stats the file for its log line; hoist that into a local and add it
to a counter:

```c
static _Atomic uint64_t g_arch_bytes;   // what the archive files hold, in all
```

```c
off_t fsz = (off_t)[[NSFileManager.defaultManager attributesOfItemAtPath:path error:nil] fileSize];
atomic_fetch_add(&g_arch_bytes, (uint64_t)fsz);
```

and the log line uses `fsz / 1048576.0` instead of asking again.

Files that predate this run are counted once, at the start of `pso_prebuild`,
by walking `g_arch_dir` (`arch_bytes_on_disk()`), so a budget applies to a
7.6 GB archive left by an earlier build as well.

### The budget

```c
static uint64_t g_pre_budget;          // DM_PSO_ARCHIVE_MAX, or the iOS default
static _Atomic int g_pre_budget_hit;
```

Read at the top of `pso_prebuild()`, after `pso4_init()` (the archive folder is
known there):

- `DM_PSO_ARCHIVE_MAX=<bytes>` when set (0 = no budget);
- otherwise on iOS `os_proc_available_memory() / 2`, clamped to 256 MiB … 2 GiB;
- otherwise (Mac) 0.

The log line ties it to the archive's current size, which is the number a bug
report needs:

```
[mem] pipeline archive: 7.61 GB on disk, budget 1.97 GB (DM_PSO_ARCHIVE_MAX unset, half the room); the prebuild stops there and leaves the rest to the draws
```

### Stopping

The workers pull pairs from `g_pre_next`. The check goes in the worker loop,
before a pair is taken:

```c
if (g_pre_budget && atomic_load(&g_arch_bytes) > g_pre_budget && !atomic_exchange(&g_pre_budget_hit, 1)) {
    atomic_store(&g_pre_next, n);      // no pair is left: the end is the end
    MLOG("pipeline prebuild: stopped at the archive budget (%.2f GB on disk, budget %.2f GB) at %d of %d pairs: the rest are the draws'", ...);
    arch_flush();
    break;
}
```

`g_pre_next = n` makes the existing end-of-prebuild path run unchanged, stamp
included, so the next start does not repeat the walk. A pair already put back
(`pre_later`) still runs to completion — a few hundred at most, and each makes
the archive smaller rather than bigger.

### Cost and risk

- The prebuild becomes a bounded job, so a device that cannot hold the whole
  archive still gets the hybrid pipelines for what it built.
- Stopping early leaves later scenes without a stand-in: a first draw of a new
  pair compiles it in the foreground (tens of ms, one stutter), as with
  `DM_PSO_PREBUILD=0` but only for the excess.
- The budget is a disk figure, not a footprint one. It bounds the file cache
  pressure, which is what `fc-thrashing` measures; it does not bound the
  archive's own mapped pages. Deliberate: the file cache is the recoverable
  half.

## D2 · Texture-view ways (R2)

`VC_WAYS = 4` is a compile-time array bound used in four places: the struct, the
free, the lookup and the replacement index.

```c
enum { VC_WAYS_DEFAULT = 4, VC_WAYS_MAX = 64, VC_MAX = 16384 };
static unsigned g_vc_ways = VC_WAYS_DEFAULT;   // DM_NVN_VC_WAYS, 1 to VC_WAYS_MAX
typedef struct { const void *pool; unsigned next; vc_key k[VC_WAYS_MAX]; void *view[VC_WAYS_MAX];
                 void *rows; uint64_t rows_rid, rows_n; } vc_set;
```

`vc_ways_init()` reads `DM_NVN_VC_WAYS` once (`pthread_once`), clamping to
1…`VC_WAYS_MAX`; `vc_get`, `vc_put` and `vc_set_free` iterate `g_vc_ways` and
`vc_put` replaces at `s->next++ % g_vc_ways`.

### Why the bound is fixed at 64 rather than allocated per set

`vc_set` grows from 128 B to about 2 KB. A title with a few thousand textures
that carry views pays a few MB — measured on the Mac in the acceptance run and
reported there rather than assumed. A variable-length set would save that and
add an allocation and a size field to three hot paths; not worth it before a
profile says the waste matters.

### Cost and risk

- More ways = more retained `MTLTexture` views: each is small (a view, not a
  copy of the texture), but it counts in the residency set.
- 64 is the value from the 8 GiB runs in `docs/reference/memory.md`; 8 is a
  cheap middle. The knob exists so this is a measurement, not a bet.

## D3 · Settings and docs

`docs/settings.md`'s env table gains:

| Setting | Variable | Values |
|---|---|---|
| Texture-view ways | `DM_NVN_VC_WAYS` | `1` to `64`, default `4` |
| Pipeline archive budget | `DM_PSO_ARCHIVE_MAX` | bytes; unset: half the room on iOS, none on the Mac |

The iPhone settings screen is unchanged: both are tuning knobs, not choices a
player makes.

## Test plan

1. Mac: `DM_NVN_VC_WAYS=64` vs unset on the village route, compare
   `texture views: … hit / made` and footprint.
2. Mac: `DM_PSO_ARCHIVE_MAX=300M` on a prebuild from an empty `pso_archive`,
   confirm the stop line, the stamp, and that the next start skips the walk.
3. iPhone: build, install, watch `[mem]` and `[mem] memory limit` for 20 min.
