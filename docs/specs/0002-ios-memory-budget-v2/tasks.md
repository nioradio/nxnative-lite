# 0002 · 任务

按依赖顺序。每个任务给出验收与回滚，未达成不做下一个。

| # | 任务 | 文件 | 验收 | 回滚 |
|---|---|---|---|---|
| T1 | 归档预算改为 `room/4`，封顶 1536 MB | `dm/gfx/nvn/nvn_mtl.m` | 日志出现 `budget 1.5 GB`；`pso_archive` ≤ 1.5 GB | `DM_PSO_ARCHIVE_MAX=0` |
| T2 | RomFS 镜像 fd 加 `F_NOCACHE` | `dm/host/romfs_image.h` | `page-ins/分钟` 下降；无 `[RED]` | `DM_FS_NOCACHE=0` |
| T3 | guest 分块按余量分档：room < 8 GB → 512 MB | `dm/host/hle_svc.c` | 日志 `N files of 0.50 GB` | `DM_GUEST_FILE_CHUNK=1G` |
| T4 | `VC_WAYS` 默认 16（小余量机型）；内存警告回收视图缓存 | `dm/gfx/nvn/nvn_mtl.m`、`dm/ios/Sources/front_ios.m` | 复用率 ≥ 90%；警告后 footprint 下降 | `DM_NVN_VC_WAYS=4` |
| T5 | 诊断：`[mem] guest file` 行加读放大比与每分钟 page-in | `dm/host/hle_svc.c` | 新字段可被正则抓到 | 纯日志 |
| T6 | 写 `~/mem-harness.sh`，3 配置 × N 轮自动跑 | 新脚本 | 产出 `~/mem-runs.csv` | — |
| T7 | Mac 测试台跑基线 + 迭代，直到 G1–G5 达成 | — | goals.md 表 | — |
| T8 | iOS 模拟器构建实测，记录失败点 | — | `simulator-report.md` | — |
| T9 | 真机构建 + 装机验证 | — | 30 分钟零 jetsam | 回退旧 IPA |

## 执行顺序

1. T1–T5 一次改完（都是小改动，互不冲突），一次编译验证
2. T6 建测试台，先测基线（改动前的行为用 env 关掉即可复现）
3. T7 逐项开关做 A/B，收敛到目标
4. T8 模拟器
5. T9 出真机 IPA，交用户执行

## 依赖与前置

- 必须先 `git pull`（用户硬规则）
- 本地已有补丁：`pack_app.py`（2 处）、`build_ipa.sh`（`-disable-sandbox`）——stash/pop，不能丢
- 构建需 `unset PYTHONPATH`、`CODEBUDDY_SAFE_DELETE_BULK_THRESHOLD=500000`
