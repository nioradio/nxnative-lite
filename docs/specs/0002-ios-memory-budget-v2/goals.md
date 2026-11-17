# 0002 · 目标与验收标准

所有指标都来自日志里已有的行，不需要新增埋点即可测量；括号内是"从哪一行读"。

## 0. 基线（实测，`b396df3a` / 链 `9eecb8412308`，iPhone 17e 8 GB）

| 指标 | 基线值 | 来源行 |
|---|---|---|
| PSO 归档体积 | **7.6 GB**（54 个 `.mtl4a`） | `pso_archive/` 文件统计 |
| 累计读入（单次会话末） | **21.29 GB** | `[mem] guest file:` 的 `process read` |
| page-ins（单次会话末） | **1,122,169** | 同上 |
| footprint 峰值 | **3.78 GB** | 同上，`peak so far` |
| 外部 clean file pages（末） | **0.01 GB**（被全数驱逐） | 同上，`file pages mapped (external)` |
| GPU allocated | 5,600 MB | `memory up to frame` |
| shared textures 常驻 | 124–196 MB | 同上 |
| 会话结局 | 07:49 / 08:07 两次 jetsam，判词 `fc-thrashing` | JetsamEvent `.ips` |

## 1. 主目标（必须全部达成）

**G1 · 归档预算生效且可控**

- OoT 全量 prebuild 后，`pso_archive/` 总大小 ≤ **1.5 GB**（基线 7.6 GB，−80%）
- 日志出现 `pipeline prebuild: stopped at the archive budget (...)`，且下一轮启动**不重跑** prebuild（stamp 生效）
- 测量：构建目录 / 设备容器内 `pso_archive` 字节求和

**G2 · 读放大收敛**

- 20 分钟会话内，`process read` ÷ guest 文件 `allocated on disk` ≤ **2.0**（基线末段约 3.2，且仍在爬）
- 稳态（最后 10 分钟）page-ins ≤ **20,000 / 分钟**（基线风暴期约 150,000 / 分钟）

**G3 · 不再被文件缓存抖动带杀**

- 30 分钟真机会话**零 jetsam**；若被终止，判词不得为 `fc-thrashing`
- 测量：`devicectl device copy from --domain-type systemCrashLogs` 后检查新 `.ips`

**G4 · 峰值 footprint**

- 同一 20 分钟脚本化场景，Mac 测试台 footprint 峰值 ≤ **3.0 GB**（仓库文档记录同设置下 4.37–4.79 GB）
- 真机 `[mem] guest file: ... peak so far` ≤ **3.2 GB**（基线 3.78）

**G5 · 视图重建不再每帧发生**

- `views reused ÷ (reused + made)` ≥ **90%**（`VC_WAYS=64` 后测）
- 测量：`texture views by Metal texture (vc_get)` 行

## 2. 次目标（达成更好，不阻塞）

**G6 · 内存警告可回撤**

- 注入一次内存警告，footprint 下降 ≥ **200 MB**，且随后 5 分钟无渲染错误
- 测量：警告前后 `[mem] guest file` 的 footprint + `[RED]` 计数

**G7 · 性能不回退**

- Mac 测试台同一场景 fps 不低于基线的 **95%**
- 真机稳态 ≥ 30 fps，稳态单帧 > 100 ms 的比例 < 5%

## 3. 验收方法

| 层级 | 怎么验 | 谁执行 |
|---|---|---|
| 代码 | `make -C dm/host/tests memory memory-file`、`make -C dm/gfx/tests` | 自动 |
| Mac 测试台 | `~/mem-harness.sh`，3 场景 × N 轮，解析日志出 CSV | 自动 |
| iOS 模拟器 | `xcodebuild -sdk iphonesimulator` 能否出可运行包（**预期失败，需记录失败点**） | 自动 |
| 真机 | 装 IPA + 推 romfs + 30 分钟脚本化操作 | 需要用户点几下 |

## 4. 本轮明确不做

- 纹理全局预算（Ryujinx 式）——改动大，且需要新的"纹理总量/命中"指标才能验收，放到 0003
- 按场景 residency set 分组——依赖游戏场景划分，难通用
- 任何降低渲染分辨率/帧率的"优化"——已确认对内存无实质帮助，且属于玩法取舍
