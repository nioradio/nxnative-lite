# 0003 · 设计

## 1. 原则

- **只在小内存机型上改默认**：判据是启动时 `os_proc_available_memory()` < 8 GB（`own_small_device()`，Mac 恒为 0/false）。Mac 的行为一行都不变。
- **每个新上限都可覆盖**：`DM_TEX_GHOST_MB` / `DM_NVN_UPLOAD_SRC_MB` / `DM_NVN_UPLOAD_IDLE_MS` / `DM_PSO_ARCHIVE_MAX` / `DM_NVN_VC_WAYS`，任一设 `0` 即回到旧行为，因此同一台手机上可以做对照。
- **不引入新的释放语义**：ghost 表的淘汰仍走 `ghost_del_locked` → `own_hook_drop`（延迟释放，跨在飞行中的提交），只是判据从"条数"扩到"字节"。
- **一句话可观测**：`[mem] small device profile …` 说明这次用的是哪一档。

## 2. 改动

### 2.1 上游 5 个修复（逐个取补丁，不动历史）

`943a6111`（`RH_SLACK` 4 MiB 封顶）、`1d1c5b4a`（零填充按行分片）、`93c76c1c`（staging 空闲归还）、`31fdefeb`（零拷贝源预算）、`4815cbbf`（iOS staging 克隆）。只取代码侧，文档/游戏笔记随下次完整合并。

### 2.2 ghost 表按字节封顶（`nvn_owner.m` / `nvn_owner.h`）

| 位置 | 内容 |
|---|---|
| `g_ghost_bytes` | 表内对象 `allocatedSize` 之和；`tex_put_locked` 增删时维护，`ghost_del_locked` 扣减 |
| `ghost_obj_bytes(id)` | package 取 `own_hook_parts` 的第一个纹理（视图与父纹理共享存储，不能重复计），否则对象自身 |
| `ghost_budget()` | `DM_TEX_GHOST_MB`；未设时，小内存机型 384 MB，其余 0（不启用字节判据） |
| `ghost_trim()` | 触发条件 `g_ghost.n > NGHOST \|\| (cap && g_ghost_bytes > cap)`；超过字节预算时把单次淘汰额度放大到 8×`GHOST_TRIM`，让远超预算的表在几个提交内下来，而不是几百个 |
| `own_ghost_bytes()` / `own_ghost_budget()` | 给日志与诊断读 |

为什么不会误杀热纹理：`ghost_trim` 仍然从 LRU 头开始，遇到**第一个未空闲**的条目就停（`GHOST_IDLE` 帧或 `GHOST_IDLE_SUBS` 次提交没用过）。正在被每帧采样的纹理会把自己顶到 MRU，永远走不到淘汰。

### 2.3 小内存机型的其余默认（`nvn_mtl.m`）

| 开关 | 原来 | 小内存机型 |
|---|---|---|
| `DM_NVN_UPLOAD_SRC_MB` | 0（关） | **96 MB** |
| `DM_NVN_UPLOAD_IDLE_MS` | 0（关） | **2000 ms** |
| `DM_PSO_ARCHIVE_MAX` | room/4，封顶 1.5 GB | room/4，封顶 **1 GB**（源码注释本来就写"过了一 GB 基本都是玩家很少走的分支"） |
| `DM_NVN_VC_WAYS` | 4 | **16**（不变，上一轮已定） |

`small_device_note()` 在 `mtl_init` 建好设备后打印一次，`vc_ways_init()` 先跑，保证打印的是实际生效值。

### 2.4 文档

`docs/env_switches.tsv` 补齐这轮与上一轮新增的 6 个开关（`DM_FS_NOCACHE` / `DM_NVN_VC_WAYS` / `DM_NVN_UPLOAD_SRC_MB` / `DM_PSO_ARCHIVE_MAX` / `DM_PSO_ARCHIVE_PRUNE` / `DM_TEX_GHOST_MB`），并修正 `DM_NVN_UPLOAD_IDLE_MS` 的默认值说明。`dm/tools/env_switches.py --check` 与 `test_env_switches.py` 通过。

## 3. 已知取舍与风险

| 风险 | 评估 | 回落 |
|---|---|---|
| 淘汰过头导致纹理重建抖动 | 只在空闲（≥16 帧未用）时淘汰；命中会重建，代价是一次上传 | `DM_TEX_GHOST_MB=0` 关闭 |
| 384 MB 低于 OoT 真实工作集，导致反复重建 | 若日志出现 `texture re-uploads` 暴涨即为此 | 调到 `768` 再测 |
| 归档 1 GB 让部分管线现场编译 | 首帧卡顿，但不影响内存判定 | `DM_PSO_ARCHIVE_MAX=1610612736` 恢复 1.5 GB |
| 这些默认基于"启动时 room < 8 GB"，room 读数受启动时序影响 | 启动早期读一次并打印，日志可直接核对 | 显式设 `DM_*` 覆盖 |

## 4. 明确不在本轮

- 纹理镜像全局预算 + LRU（要动 `NvPkg` 生命周期，风险高，需真机数据支撑）。
- 小内存机型降内部分辨率 + MetalFX 上采样（上游 PR #2270 仍未合并；它是唯一能按比例砍 2.1 GB 常驻 target 的手段，下一轮评估）。
- 完整合并上游 101 个提交：网络 26 KB/s，改动主体是各游戏几百 MB 的着色器表，与 OoT 无关；代码侧需要的 5 个已逐个取证。
