# 0003 · 目标与验收标准

指标全部来自已有的日志行，不需要新埋点；括号内写明读哪一行。

## 1. 基线（iPhone 17e，8 GB，2026-11-17 实测）

| 指标 | 基线 | 读法 |
|---|---|---|
| footprint 峰值 | **3.78 GB** | `[mem] guest file: … peak so far` |
| 会话结局 | 4 次 jetsam（`fc-thrashing` / `vm-pageshortage`） | `JetsamEvent` `.ips` |
| 读放大 | ≈2.8×（21.29 GB / 7.6 GB） | `process read … (…x what is allocated)` |
| page-in | 稳态风暴期约 150,000/分钟 | 同上括号内 |
| PSO 归档 | 7.6 GB | 设备容器 `Documents/pso_archive` 求和 |
| 帧率 | 30 fps 档，p50 33.3 ms | `frame …p50` 行 |

## 2. 主目标（必须达成）

**G1 · 机型档生效且可见**
- 日志出现恰好一条 `[mem] small device profile (room at start … MB): …`
- 各值与设计一致：源预算 **96 MB**、staging 空闲 **2000 ms**、ghost 上限 **384 MB**、视图 **16** 路
- 同一份包在 Mac 上不出现该行（Mac 不受影响）

**G2 · ghost 表有确定性上限**
- `memory up to frame …: … shared textures %u (… %llu MB …)` 里那个 MB 数在同一次会话内**回落到 ≤ 384 MB**（可以是工作集之上），且不再单调上升
- 不发生因此产生的渲染错误：`[RED] released too early` 计数为 0

**G3 · 30 分钟不被杀**
- 一次连续会话 ≥ 30 分钟，**零 jetsam**；若仍被杀，判词中不得为 `fc-thrashing`

**G4 · 峰值 footprint**
- 同一路线（标题 → 时之神殿存档 → 走出村庄 → 卡卡利科载入），`peak so far` ≤ **3.4 GB**（基线 3.78）

**G5 · 读放大与 page-in 收敛**
- `process read ÷ allocated` ≤ **2.0**
- 稳态（最后 10 分钟）page-in ≤ **20,000/分钟**

## 3. 次目标（不阻塞）

**G6 · 帧率不回退**
- 30 fps 档 p50 不高于基线 +10%（≤ 36.6 ms）；`GPU p50` 不劣于基线

**G7 · staging 与源缓冲可见地下降**
- `DM_NVN_UPLOAD_IDLE_MS=0` 与默认对照，日志里 staging 常驻 MiB 明显更低

## 4. 验收方法

| 层 | 怎么做 | 谁 |
|---|---|---|
| 编译 | `scripts/build_ipa_from_nsp.py` 出包，`codesign --verify` 通过 | 本机 |
| 装机 | Impactor 拖包覆盖安装（存档不丢） | 用户（免费签名，必须人工） |
| 对照 | `~/ios-harness.sh` 按 `Documents/env.txt` 跑轮次，自动拉回 `log.txt` | 本机 |
| 判定 | 只看上面列出的日志行与 `JetsamEvent`，不看感觉 | 本机 |

**A/B 约定**：每轮只改一个变量，同一存档、同一路线、同一热状态；先跑默认（新档），再跑 `base`（把新开关逐个关掉：`DM_TEX_GHOST_MB=0 DM_NVN_UPLOAD_SRC_MB=0 DM_NVN_UPLOAD_IDLE_MS=0`）。
