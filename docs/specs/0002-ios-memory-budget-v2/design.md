# 0002 · 设计

## 总原则

只做"能减少文件缓存需求"或"能确定性回收内存"的改动，每处都留可回滚开关，每处都必须能在 Mac 测试台上量化。

## D1 · PSO 归档预算（修订 0001 的 T2）

**现状**：`pso_prebuild()` 已累加 `g_arch_bytes`，超预算即停。iOS 默认 `room / 2`（本机 ≈ 3 GB），封顶 2 GB。

**问题**：3 GB 仍然过大。基线里 7.6 GB 归档是 `fc-thrashing` 的头号贡献者，压到 3 GB 只是"减半"，不是"收敛"。且 OoT 的 base 管线在 1 GB 内基本覆盖（BotW 全量 15,767 对约 850 MB）。

**改法**：

```c
g_pre_budget = room / 4;                       // 原 room / 2
clamp: 256 MB .. 1536 MB                        // 原 256 MB .. 2 GB
```

理由：`room/4` 在 8 GB 机型上给出 ≈1.5 GB，正好等于目标上限；在 12 GB 机型上给出 ≈1.5 GB（封顶），一致性好过按比例放大——归档的边际收益递减，超过 1.5 GB 的部分几乎不命中。

**风险**：未进归档的管线要在首次使用时现场建（几十 ms，可能丢一帧）。可接受：`fc-thrashing` 的代价是 11 fps 和被杀。

**回滚**：`DM_PSO_ARCHIVE_MAX=0` → `g_pre_budget=0` → 无预算（原行为）。

## D2 · RomFS 镜像读取 `F_NOCACHE`（新增）

**现状**：`romfs_image_open()` 打开镜像后，所有 `ReadFile` 都走 `pread` 到 `img->fd`。24.4 GB 的纹理解压数据因此进入 unified buffer cache，随后驱逐 guest 文件页。

**改法**：镜像打开后对 fd 设 `F_NOCACHE`：

```c
#include <fcntl.h>
fcntl(fd, F_NOCACHE, 1);
```

只对 **romfs 镜像 fd** 设，不影响存档、日志、guest 文件。开关 `DM_FS_NOCACHE`（默认 1）。

**为什么不是"读一次就没人再用"**：RomFS 里的资源在游戏生命周期内基本只读一次（进场景时解压/上传到 guest 内存），之后 GPU 从 guest 内存或 Metal 纹理取数，不再回读文件。所以缓存它们没有收益，只有驱逐别人的代价。

**风险**：若某游戏反复读同一文件（例如流式加载的音频包），会退化为每次真实 I/O。用 `DM_FS_NOCACHE=0` 可关。验证指标是 page-in 次数与读入量，回退明显就能发现。

**回滚**：`DM_FS_NOCACHE=0`。

## D3 · guest 文件分块按余量分档（新增）

**现状**：默认固定 1 GB。

**改法**（`hle_svc.c`）：

```c
if (!chunk) chunk = (room < (8u << 30)) ? (size_t)512 << 20 : (size_t)1 << 30;
```

依据：仓库文档记录的 8 GiB iPhone 18 Pro（#1184）配方是 512 MB 文件 + `VC_WAYS=64`，13 分钟两场比赛、峰值 4.41 GB；且 flush 停顿随文件大小增长（1 GB 文件 4 ms，20 GB 单文件 148 ms）。

**风险**：文件数从 13 增到 25，`GF_MAX=1024` 够用；元数据开销可忽略。

**回滚**：`DM_GUEST_FILE_CHUNK=1G`。

## D4 · `VC_WAYS` 运行时化（0001 的 T1，保留）

已实现：`DM_NVN_VC_WAYS`，范围 1–64，默认 4。本轮把 8 GB 机型的默认值改为 **16**（保守：64 路 × 每路一个 MTLTexture view，在 4,500 个 pool 的规模下常驻可观；16 路能覆盖绝大多数重复采样，内存代价可控）。

依据：视图复用率是可直接读的指标（`views reused / made`），先取 16 测，若复用率仍低再加。

## D5 · 内存警告回收视图缓存（新增，低风险）

**做法**：iOS 注册 `UIApplicationDidReceiveMemoryWarningNotification`，只置一个 `_Atomic int g_memwarn`；在 `vc_put()` 顶部检查并清空 `g_vc`。选择 `vc_put` 的理由：它本来就在容量上限时调用 `nvn_hm_clear(&g_vc, vc_set_free)`，同一锁上下文，线程安全性由既有代码保证。

**只回收视图缓存**，不碰 pool、不碰正在绘制用的纹理、不碰归档映射——避免任何"警告一来就渲染错"的可能。

**度量**：警告前后 footprint 差、`[RED]` 计数。

## D6 · 诊断增强（只加日志，不改行为）

在 `[mem] guest file` 行追加**读放大比**（`read / allocated`）与**每分钟 page-in**，便于测试脚本直接抓，不用二次计算。

## 测试台设计

### 为什么用 Mac

- `dm/gfx/nvn/nvn_mtl.m`、`hle_svc.c`、`hle_fs.c` 在 Mac 与 iOS 是**同一份源码、同一套 Metal 4 路径**；差异只在 `TARGET_OS_IPHONE` 分支。
- 仓库文档记录：16 GB M5 Mac 用 iPhone 设置跑 OoT 峰值 4,365–4,792 MiB，与真机同量级 → Mac 的 footprint 变化可以外推到真机。
- 无需签名、无需推 24 GB 数据，可以跑 N 轮。

### 场景与采样

| 场景 | 做法 | 采样 |
|---|---|---|
| S1 启动到标题 | 启动后静置 120 s | 每 10 s 的 guest file 行 |
| S2 进游戏固定区域 | 静置 600 s | 同上 + 每 300 帧的 memory 行 |
| S3 场景切换 | 反复进出（脚本模拟输入，若不可行则跳过并记录） | 同上 |

### 自动化

`~/mem-harness.sh`：

```
for cfg in baseline pso_cap nocache chunk512 vc16 ...
  for round in 1..N
    launch app with env
    wait T
    SIGTERM
    parse log → csv row
```

指标抽取用正则，落 `~/mem-runs.csv`，最后出一张对比表。判定用 goals.md 的阈值。

### iOS 模拟器

预期不可用，但必须实测并记录失败点：

- `project.yml` 只有 device 配置，`Gen/src` 由设备端编译命令生成
- 即使构建成功，模拟器的 Metal 4 与真机不同、无 jetsam、内存模型完全不同 → **不作为内存结论的依据**，只能验证"能否编译/启动"

结论写入 `simulator-report.md`。

## 风险与边界

1. 三项 P0 都只影响"文件缓存在哪里花"，不改变游戏可见行为；最坏情况是加载变慢或偶发一帧卡顿。
2. 所有改动不动 guest 布局、不动渲染结果，**不会引入画面错误**。
3. 真机验证仍需用户操作 30 分钟；Mac 结论只能到"趋势正确、量级可信"。
