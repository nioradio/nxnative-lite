# 0003 · 任务

## 已完成

- [x] 建分支 `oot-iphone17e`（基于 lite `main` = 上游 `3f54c610` + 上一轮内存改动）
- [x] 逐个取上游 5 个内存修复（`943a6111` / `1d1c5b4a` / `93c76c1c` / `31fdefeb` / `4815cbbf`），只取代码侧
- [x] `nvn_owner`：ghost 表字节记账 + 字节预算 + 淘汰判据扩展
- [x] `nvn_mtl`：小内存机型的四个默认（源预算 96 MB / staging 空闲 2 s / 归档 1 GB / 视图 16 路）+ 启动一行说明
- [x] 两个 .m 用真实编译参数（`dm/host/Makefile` 的 `HOST_CFLAGS`）编译通过
- [x] `docs/env_switches.tsv` 补齐 6 个开关，`env_switches.py --check` 与 `test_env_switches.py` 通过
- [x] 出包：`ZeldaOcarinaOfTime-8gb.ipa`

## 待真机（需要 USB 线，设备当前 `unavailable`）

- [ ] Impactor 拖包覆盖安装（存档不丢）
- [ ] 首次启动：确认 `[mem] small device profile …` 一行存在且数值符合设计（G1）
- [ ] 跑默认档 30 分钟：零 jetsam（G3）、`peak so far` ≤ 3.4 GB（G4）、读放大 ≤ 2.0（G5）
- [ ] 跑对照档（`env.txt` 里 `DM_TEX_GHOST_MB=0 DM_NVN_UPLOAD_SRC_MB=0 DM_NVN_UPLOAD_IDLE_MS=0`）同样路线
- [ ] 看 `shared textures … MB` 是否回落到 ≤ 384 MB（G2），`texture re-uploads` 是否暴涨（若是，把 384 调到 768 再测）
- [ ] 帧率不回退（G6）

## 下一轮（需要本轮数据）

- [ ] 若 ghost 封顶不足：纹理镜像全局预算 + LRU（`NvPkg` 生命周期）
- [ ] 评估把上游 PR #1971 / #2270（小内存机型降内部分辨率 + MetalFX 上采样、空闲纹理 10 s 清理）合入 lite 的可行性与冲突面
- [ ] 网络恢复后做一次真正的上游合并，把 `docs/` 与游戏笔记补齐
