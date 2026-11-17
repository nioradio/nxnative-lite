# 0002 · iOS 内存与文件缓存预算：调研

状态：取代 `0001-ios-memory-budget`。0001 的诊断（"guest 工作集太大"）被本轮的实测数据证伪：guest 早已在 sparse file 里，footprint 只有 0.45–1.0 GB；真正把 app 推出去的是 **host 侧与文件缓存**。本文件先给证据，再给外部调研，最后落到候选手段。

---

## 1. 现场证据（本机实测，不是推断）

### 1.1 设备与配置

| 项 | 值 |
|---|---|
| 设备 | iPhone 17e（iPhone18,5），A19，**8 GB**，iOS 27.2.1 |
| 构建 | `b396df3a` / 着色器链 `9eecb8412308`，`--romfs separate --shader-files` |
| 签名 | Impactor 免费 Apple ID，`com.apple.developer.kernel.increased-memory-limit` 已确认写入 amfid 序列化结果 |
| 游戏数据 | RomFS 26,247,576,430 字节（24.4 GiB）经 AFC 推送，字节数一致 |

### 1.2 启动时 guest 已在文件里（0001 的前提不成立）

```
[mem] guest memory: memory pool 9.00 GB and the host's margin 4.50 GB (half the pool),
      room to the memory limit 5.97 GB, DM_GUEST_FILE unset: in files
[mem] guest memory: 13 files of 1.00 GB in .../tmp/ (unlinked, sparse, shared),
      12.50 GB in all ... holes punched where the game gives memory back: on
```

也就是说 `guest_file_wanted()` 的规则已经生效：9 GB pool + 4.5 GB margin > 5.97 GB 余量，guest 落进 13 个 1 GB 的 sparse 文件。**guest 的常驻页不计 footprint**。

### 1.3 但 page-in 风暴出现了

同一份日志，按时间顺序取三次 `[mem] guest file` 采样：

| 采样 | 进程累计读入 | page-ins | footprint（峰值） | 其中压缩 | 外部 file pages |
|---|---|---|---|---|---|
| 早 | 7.15 GB | 529,745 | 1.00 GB (1.07) | 0.43 GB | 1.37 GB |
| 中 | 7.68 GB | 617,560 | 0.90 GB (1.07) | 0.43 GB | 1.99 GB |
| 末 | **21.29 GB** | **1,122,169** | **3.11 GB (3.78)** | **0.78 GB** | **0.01 GB** |

两点关键：

1. **末段在极短时间内多读了 13.6 GB**，而游戏并没有加载新场景——这是 clean 的 guest 文件页被内核回收后，游戏再次触碰同一批地址导致的**重读**。
2. `file pages mapped (external)` 从 1.99 GB 掉到 0.01 GB，说明内核把 clean 页全数驱逐了。驱逐的原因只能是**文件缓存不够**。

### 1.4 文件缓存的竞争者

| 竞争者 | 体积 | 性质 |
|---|---|---|
| RomFS 镜像 | 24.4 GB | 逐文件 `pread`，一次性读取后即无用，但会留在 unified buffer cache |
| PSO 归档 `pso_archive/` | **7.6 GB**（54 个 `.mtl4a`） | `newArchiveWithURL` mmap，clean pages，不进 footprint 但**占 file cache** |
| guest 文件 | 12.5 GB 逻辑 / 6.75 GB 已分配 | clean 页被驱逐就要重读 |

三者相加远超 8 GB 物理内存能提供的文件缓存。系统此时的选择是回收 clean 页，于是 guest 文件被反复读回——这正是 jetsam 判词 `fc-thrashing`。

### 1.5 jetsam 判词（四份报告）

| 时间 | DmIOS rpages | 系统判词 | 同时被牵连的进程 |
|---|---|---|---|
| 00:53 | 1.04 GB | jetsam | — |
| 01:03 | 3.96 GB | `vm-pageshortage` | — |
| 07:49 | 3.48 GB | jetsam | — |
| **08:07** | **3.86 GB** | **`fc-thrashing`** | 抖音 0.95 GB（`vm-pageshortage`）、微信 0.63 GB（`fc-thrashing`） |

`fc-thrashing` 与 `vm-pageshortage` 都是**全机状态**，不是"本进程超过自己的额度"。这解释了为什么 `increased-memory-limit` 没能救场：它抬的是单进程额度，管不了全机文件缓存。

### 1.6 host 侧占用（不是 guest）

```
memory up to frame 3600: resident 3511 MB, GPU allocated 5600 MB, ...
memory pools 4545 (resident 4305; peak 4564 ...), shared textures 241 (... 124 MB ...)
```

shared textures 只占 124 MB，`GPU allocated` 里含零拷贝的 guest pool 缓冲（文档明确说明这个数字不能当 footprint 用）。因此 resident 的 3.5 GB 里，大头是 **guest 文件的 clean 页 + Metal 自己的纹理/渲染目标**，而不是"游戏堆太大"。

---

## 2. iOS 内存计量模型（外部实证）

这一节决定优化方向，必须查清而不是猜。

### 2.1 jetsam 只看 footprint，clean 文件页不计入

> "Clean memory: Pages backed by files on disk (code, mapped resources). Can be evicted and re-loaded from disk. The number that Jetsam cares about is your process's physical footprint — **dirty + compressed + wired**. Clean pages aren't counted because they can be reclaimed for free by re-reading from disk."
>
> — *Memory: Jetsam, Compression, and No Second Chances*, amgres.com

推论一：把数据挪到 file-backed mapping 能降低 footprint（这正是本项目 guest file 的原理）。
推论二：**但"不计入 footprint"不等于"免费"**——它把成本从 footprint 转移到了文件缓存与 page-in。当文件缓存不够时，代价以 `fc-thrashing` 与掉帧的形式回来。本项目正落在推论二里。

### 2.2 8 GB 机型的实际上限

| 来源 | 8 GB 机型 + entitlement | 备注 |
|---|---|---|
| gemma-4.run 实测表 | 约 4.8–5.0 GB/进程 | iPhone 15 Pro/16（A17/A18） |
| bigiron.cc | 4–5 GB | 并注明"需要 increased-memory-limit，无此权限更低" |
| 本项目 `docs/reference/memory.md` | 12 GB 机型 → **6,143 MiB**（LiveContainer 实测） | 按比例折算 8 GB ≈ 4 GB 上下 |
| 无 entitlement 的 8 GB | 约 3.2–3.5 GB | 同上来源 |

本机实测的 room 是 **5.97 GB**（启动时 `os_proc_available_memory()`），与"4.8–5.0 GB 可用 + 若干可回收"吻合。所以额度不是瓶颈，**文件缓存才是**。

### 2.3 压缩器是压力的指示器

> "iOS does use memory compression — just not swap. ... If compressed pages plus active pages fill physical RAM, Jetsam kills processes to free both."
>
> — amgres.com

本机末段 compressed 已达 **0.78 GB**，且当天累计 909 万次压缩 / 760 万次解压。这是"内存已经不够、正在用 CPU 换空间"的直接证据。

---

## 3. 同类引擎的成熟做法

### 3.1 Ryujinx：给纹理缓存设硬上限 + LRU

`AutoDeleteCache.cs` 的实测参数：

```csharp
private const int MinCountForDeletion = 32;
private const int MaxCapacity = 2048;
private ulong MaxTextureSizeCapacity = 4 * GiB;
private const ulong MinTextureSizeCapacity = 512 * MiB;
private const float MemoryScaleFactor = 0.50f;   // 缓存在可用显存里的占比

// 按物理内存分档
cpuMemorySizeGiB >= 8  → MaxTextureSizeCapacity = 6 GiB
cpuMemorySizeGiB == 6  → 4 GiB
cpuMemorySizeGiB <= 4  → 2 GiB

_maxCacheMemoryUsage = Clamp(MaximumGpuMemory * 0.5, 512 MiB, MaxTextureSizeCapacity);
```

要点：**上限按设备物理内存分档**，且按"可用显存的一半"计算，而不是固定值；超出就 LRU 淘汰，并保证至少留 32 条不被删（避免抖动）。

→ 本项目当前**没有**这个机制：纹理随 pool 生命周期走，没有"全局纹理预算"。

### 3.2 yuzu：page table + LRU 垃圾回收

- `PageTable`：`DAddr >> PAGEBITS` 映射到 image 列表，CPU 写时精确失效
- `LRU Cache`：按访问模式回收长期未用的 image
- `Slot Vectors`：固定容量、索引稳定，删除后槽位复用

→ 本项目有 page table（`nvn_dirty.c` 的写保护 + 失效）与 view 缓存，但 view 缓存只有 4 路（见 4.4）。

### 3.3 Ryujinx PPTC：只保留"真的用到"的条目

"Profiled" 的含义是分析哪些 shader 实际需要、删掉不需要的，避免缓存被无用数据撑爆。

→ 直接对应本项目的 PSO 归档：34,649 对全建 = 7.6 GB，其中大量是玩家永远走不到的对手。

### 3.4 Apple Metal 4 的三件工具（WWDC25-205 / Metal 文档）

1. **Residency set 分组 + `endResidency()`**
   > "When your app no longer needs a residency set's allocations ... call `endResidency()`, which effectively releases them. The method tells Metal that it can reuse the memory backing that residency set's allocations for your app's other residency sets, or for another app."

   即：可以按场景（菜单/室内/平原）切换驻留集，切走的那组内存**可以被回收**。Control Ultimate Edition 用它降低了关闭光追时的内存占用。

2. **Placement sparse resource**
   > "Metal 4 supports buffers and textures allocated as placement sparse resources. These resources are allocated without any pages to store their data. ... pages come from a placement heap."

   即：纹理可以"按需给页"，是真正能压驻留量的手段，而不是靠猜。

3. **MTL4Compiler 多线程编译**
   多线程把编译任务分散，降低单线程峰值与等待。

补充（WWDC23-10125）：只读资源集中放进 heap + 每次 `useHeap` 一次；可写资源单独分配。前者降 CPU 开销，后者让 Metal 管同步。

→ 本项目已用 Metal 4（`MTL4CommandQueue`）与 residency set，但**没有按场景分组，也没有 `endResidency`**；也没有用 placement sparse。

### 3.5 iOS 的标准动作：内存警告

`UIApplication.didReceiveMemoryWarningNotification` 是系统在杀进程前给的最后机会，标准做法是丢弃可重建的缓存（图片、GPU 资源、字体）。本项目目前未注册该通知。

---

## 4. 本项目现状（代码事实，逐条核对过）

| 能力 | 现状 | 位置 |
|---|---|---|
| guest 落 sparse file | **已实现**，按 pool+margin 与 room 比较自动决策 | `hle_svc.c` `guest_file_wanted` |
| 归还内存打洞 | **已实现**，`guest_release` → `PROT_NONE` + punch hole | `hle_svc.c` `guest_file_punch` |
| guest 文件分块 | 默认 **1 GB**（`DM_GUEST_FILE_CHUNK`） | `hle_svc.c` |
| PSO 归档体积 | 无上限，OoT 实测 **7.6 GB** | `nvn_mtl.m` `arch_write_file` |
| RomFS 读取 | **逐文件 `pread`**，无 `F_NOCACHE` | `hle_fs.c:1104` |
| 纹理视图缓存 | `VC_WAYS = 4` 硬编码（已在本轮改为可调，默认仍 4） | `nvn_mtl.m:6936` |
| 纹理全局预算 | **无**（无 Ryujinx 式上限） | — |
| 内存警告处理 | **未注册** | — |
| 按场景切换驻留集 | 有 residency set，但无分组、无 `endResidency` | `nvn_mtl.m` |
| 诊断指标 | **齐全**：resident / GPU allocated / pools / shared textures / view 复用 / page-in / footprint 分类 | `nvn_mtl.m:5185`、`hle_svc.c` |

诊断指标齐全这一点很重要：它让"改完是否有效"可以量化，而不是凭感觉。

---

## 5. 差距分析 → 候选手段

排序依据：**（预期收益 × 可在 Mac 上量化验证）÷ 风险**。

| # | 手段 | 针对的病灶 | 预期收益 | 风险 | 验证方式 |
|---|---|---|---|---|---|
| P0-1 | PSO 归档字节预算（默认 iOS 取 room/2，封顶 2 GB） | 7.6 GB 归档吃 file cache | 释放 **5–6.5 GB** file cache | 新管线现场建，偶发一帧卡顿 | 归档目录大小 + `stopped at the archive budget` 日志 |
| P0-2 | RomFS/game-data 读取加 `F_NOCACHE` | 24 GB 一次性读污染 file cache | 保护 guest 文件页不被驱逐 | 若游戏重复读同一文件会变慢 | page-in 次数/分钟、读入 GB/分钟 |
| P0-3 | guest 文件分块 1 GB → 512 MB（≤8 GB 机型） | 每 30 s 的 flush 停顿、驱逐粒度 | 停顿从 148 ms 级降到 4 ms 级 | 文件数翻倍（13 → 25） | 日志里的 flush 停顿与 page-in 曲线 |
| P1-1 | `VC_WAYS` 可调，8 GB 机型用 64 | 每帧 view 重建 | view 重建降一个数量级 | 常驻 +几十 MB | `views reused / made` 比值 |
| P1-2 | 注册内存警告 → 丢弃可重建缓存（idle shared textures、view 缓存、归档句柄） | 被 jetsam 前的最后窗口 | 峰值回撤数百 MB | 需保证不丢正在用的资源 | 注入测试：手动发内存警告看回撤 |
| P2-1 | 纹理全局预算（Ryujinx 式：room × 0.5，分档，LRU） | host 侧纹理长期增长 | 理论上限可控 | **改动大**，可能淘汰热纹理导致卡顿 | 需要新指标：纹理总量/命中率 |
| P2-2 | 按场景 residency set 分组 + `endResidency` | 场景切换后旧场景内存滞留 | 数百 MB | 依赖游戏场景划分，难通用 | 场景切换前后 footprint |

**不做**（本轮排除）：

- 改 guest 到匿名内存：会让 9 GB pool 直接进 footprint，必死。
- 缩渲染分辨率：只省 100–200 MB，对 3.8 GB 的缺口无意义，且已确认 30 fps + Handheld 是玩法层面的取舍，与内存无关。
- 关闭 `Frame buffering`：省不到 10 MB。

---

## 6. 结论

1. 0001 的方向错了：guest 不是问题，**host 侧与文件缓存**才是。
2. 最有效的三件事都在"减少 file cache 需求"上，且都不需要 GPU 侧重构：归档预算、RomFS 不污染缓存、guest 分块变小。
3. 纹理全局预算（P2-1）是唯一能给出**确定性上限**的手段，但改动大、需要新指标，放在 P0 验证完之后。
4. 所有手段都可以在 Mac 上用 `DM_GUEST_FILE=1` + iPhone 同等配置量化：仓库文档记录过 16 GB M5 Mac 用 iPhone 设置峰值 4,365–4,792 MiB，与真机同量级。

## 参考

1. *Memory: Jetsam, Compression, and No Second Chances* — https://amgres.com/blog/ios-memory-jetsam-flutter
2. *Can You Run Gemma 4 on iPhone and iPad*（8 GB 机型 jetsam 上限实测表）— https://gemma-4.run/blog/can-you-run-gemma-4-on-iphone
3. *LLMs on an iPhone or iPad*（8 GB ≈ 4–5 GB，12 GB ≈ 9–10 GB）— http://www.bigiron.cc/guides/llms-on-an-iphone-or-ipad-mlx-yes-neural-engine-no
4. *JetsamEvent 日志解读*（机型 × 前后台阈值表）— https://github.com/zhwayne/awesome-ios-interview/blob/main/articles/ios-advanced/崩溃/JetsamEvent日志解读.md
5. Ryujinx `AutoDeleteCache.cs` — https://git.ryujinx.app/kenji-nx/ryujinx/-/blob/a58295b88588509bc1be8cbeb16c4a6eaf010229/src/Ryujinx.Graphics.Gpu/Image/AutoDeleteCache.cs
6. Ryujinx PPTC 指南 — https://ryujinx.io/?p=2815/
7. yuzu Texture Cache 架构 — https://deepwiki.com/exverge-0/yuzu-EA4176/4.4.1-texture-cache-architecture
8. *Discover Metal 4*（WWDC25-205，residency set / placement sparse / MTL4Compiler）— https://developer.apple.com/videos/play/wwdc2025/205/
9. *Simplifying GPU resource management with residency sets* — https://developer.apple.com/documentation/metal/simplifying-gpu-resource-management-with-residency-sets
10. *Bring your game to Mac, Part 3: Render with Metal*（WWDC23-10125）— https://developer.apple.com/videos/play/wwdc2023/10125/
11. 本项目 `docs/reference/memory.md`（guest file 规则、12 GB 机型 6,143 MiB、#1184 的 8 GiB 实测）
