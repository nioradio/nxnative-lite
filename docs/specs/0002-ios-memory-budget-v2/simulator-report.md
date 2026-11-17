# iOS 模拟器可行性：实测结论

结论：**本渲染器无法在 iOS 模拟器上构建，"在模拟器跑 N 轮"这条路结构性不可用**。下面是实测证据，不是推断。

## 1. 直接编译测试

用模拟器 SDK 编译渲染核心：

```
xcrun clang -fsyntax-only -fobjc-arc -target arm64-apple-ios26.0-simulator \
  -isysroot <iPhoneSimulator27.2.sdk> ... dm/gfx/nvn/nvn_mtl.m
```

结果：

```
nvn_mtl.m:5:9: fatal error: 'MetalFX/MetalFX.h' file not found
```

## 2. 根因：MetalFX 只随设备 SDK 发布

| SDK | `System/Library/Frameworks/MetalFX.framework` |
|---|---|
| iPhoneSimulator27.2.sdk | **不存在**（计数 0） |
| iPhoneOS26.x.sdk（设备） | 存在（含 `MTL4FXFrameInterpolator.h`、`MTL4FXSpatialScaler.h` 等） |

本项目把 MetalFX 作为放大路径（`Upscaling` 设置项）无条件编入，没有可关闭的条件编译分支。因此模拟器目标在**预处理阶段就断**，连链接都到不了。

## 3. 次要阻碍（即使补上 MetalFX 也会挡路）

1. `dm/ios/tools/gen_build.py` 把 SDK 写死为 `xcrun --sdk iphoneos`，生成的 `Gen/src` 全是设备端编译命令；`Gen/DmIOS.xcodeproj` 是构建期产物，平时不存在。
2. `dm/ios/project.yml` 只有 `platform: iOS`，没有模拟器 destination 与 `EXCLUDED_ARCHS` 配置。
3. 即便产物能跑，模拟器内存模型与设备完全不同：没有 jetsam、进程就是 macOS 进程、文件缓存由宿主 macOS 管理。**在本项目关心的指标（footprint 分类、`fc-thrashing`、page-in 行为）上，模拟器的数字没有外推价值。**

## 4. 因此采用的验证链

| 层 | 工具 | 能证明什么 | 不能证明什么 |
|---|---|---|---|
| 单元 | `make -C dm/host/tests memory memory-file`、`make -C dm/gfx/tests` | guest 文件规则、内存 SVC、图形回归 | 真机行为 |
| Mac 实机 | `~/mem-harness.sh`（同一份 `nvn_mtl.m` / `hle_svc.c`，同一套 Metal 4 路径） | 归档体积、page-in 率、footprint 变化方向与量级、视图复用率、无崩溃 | 8 GB 真机的 jetsam 判定（Mac 可换页） |
| 真机 | IPA + 30 分钟脚本化 | 是否仍被 `fc-thrashing` 终止 | 需要人工操作 |

仓库文档本身给过外推依据：16 GB M5 Mac 用 iPhone 设置跑 OoT，峰值 4,365–4,792 MiB，与 12 GB iPhone 同量级。所以 Mac 的**相对变化**可信，**绝对阈值**仍以真机为准。
