# Adreno™ Neural Fusion (ANF)

| [English](./README.md) | [简体中文](./README-zh-CN.md) |
|---|---|

每一代产品迭代，都对移动端游戏提出更高要求。玩家期望图像清晰、运动流畅，同时不牺牲电池续航，而开发者必须将游戏体验控制在移动设备的功耗预算之内。

Adreno™ Neural Fusion 是高通面向移动游戏推出的神经渲染 SDK。它为游戏团队提供了更大的空间来提升视觉质量和显示帧率，同时将功耗控制在合理范围内。应用程序选择其图像尺寸、渲染顺序和回退行为。ANF 在运行时报告硬件支持和 Vulkan 要求，以便渲染器能够为每个设备配置正确的路径。

<p align="center">
  <img src="media/anf-hero.png" width="760" alt="Adreno™ Neural Fusion 渲染的 Sanctuary 场景">
</p>

## 目录

- [ 资源 ](#resources)
- [ 技术 ](#techniques)
- [ 要求 ](#requirements)
- [ 包内容 ](#package-contents)
- [ 入门 ](#getting-started)
- [ 调试 SR 集成 ](#debugging-sr-integration)
- [ 贡献 ](#contributing)
- [ 许可证 ](#license)

## 资源

| 资源 | 用于 |
|---|---|
| [ SDK 集成指南 ](./INTEGRATION-GUIDE.md) | 本地 Android Vulkan 设置、渲染要求、技术创建、调度、同步、验证和发布检查 |
| [ 调试覆盖层指南 ](./DEBUG-OVERLAY.md) | 验证 SR 颜色、深度、运动矢量、抖动、扭曲预测和重投影 |
| [ Snapdragon™ 分析器 ](https://www.qualcomm.com/developer/software/snapdragon-profiler) | 在 Snapdragon™ 设备上性能分析和剖析支持 ANF 的应用程序 |
| [ ANF Vulkan 示例 ](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/tree/main/samples/anf) | 基于 Snapdragon™ Game Studios 示例框架构建的原生 Vulkan 集成 |
| [ 适用于 Unreal Engine 的 ANF 插件 ](https://github.com/SnapdragonGameStudios/snapdragon-game-plugins-for-unreal-engine#adreno-neural-fusion) | 将 ANF 集成到虚幻引擎项目中 |
| [ 适用于 Unity 的 ANF 插件 ](https://github.com/SnapdragonGameStudios/com.qualcomm.snapdragon.adreno.neural.fusion) | 将 ANF 集成到 Unity 项目中 |

## 技术

### 超分辨率

SR 将场景重构为应用程序所选的输出分辨率。渲染器提供经过抖动的场景颜色和深度、不含投影抖动的运动矢量，以及当前抖动偏移量。在 SR 之后合成 UI 和其他屏幕空间元素，以确保其保持清晰，并避开时间历史。

关于渲染规范和所需输入，请参阅 [ 超分辨率资源流 ](./INTEGRATION-GUIDE.md#super-resolution-resource-flow)。

### 帧生成

FG 使用连续渲染的场景帧、深度和运动矢量来创建中间场景帧。针对渲染帧和生成帧，需单独渲染或合成 UI。

关于输入、输出和合成要求，请参阅 [ 帧生成资源流 ](./INTEGRATION-GUIDE.md#frame-generation-resource-flow)。

<!-- IMAGE PLACEHOLDER
Simplified renderer-flow diagram showing:
- SR scene inputs to reconstructed scene output
- rendered scene frames to an FG-generated scene frame
- UI composed after the technique output
Use the same resource names and pass ordering as the SDK integration guide.
-->

## 要求

原生SDK集成要求：

- 为 `arm64-v8a` 构建的 Android 应用程序。
- Snapdragon™ 第 8 代 Elite Gen 6 及更高版本（计划在更远的平台路线图下提供更广泛的支持）。
- Vulkan 渲染器。
- 请求的技术不可用时的回退路径。

[ SDK 集成指南 ](./INTEGRATION-GUIDE.md) 涵盖初始化、运行时支持检查和完整的渲染契约。

## 包内容

| 路径 | 目录 |
|---|---|
| `lib/arm64-v8a/libanf.so` | Android ARM64 ANF 运行时库 |
| `public/anf.h` | 公共 API 入口点和函数表 |
| `public/anf_types.h` | 通用 ANF 类型、结构体、标志位和枚举 |
| `public/anf_types_vk.h` | Vulkan 专用 ANF 类型 |
| `public/anf_sr.h` | SR 创建和调度结构体 |
| `public/anf_fg.h` | FG 创建和调度结构体 |
| `DEBUG-OVERLAY.md` | SR 调试覆盖层文档 |
| `INTEGRATION-GUIDE.md` | 原生 Vulkan 集成与渲染契约 |
| `LICENSE-BSD-3-Clause.txt` | 公共头文件的 BSD 3-Clause 条款 |
| `LICENSE.txt` | `libanf.so` 的 QTI 免登录二进制许可条款 |

## 入门指南

ANF 既可以直接集成到原生 Vulkan 渲染器中，也可以通过虚幻引擎 和 Unity 插件进行集成。若进行引擎集成，请参照 [ 资源 ](#resources) 下列出的插件仓库中的说明。以下步骤涵盖原生 Vulkan SDK 的相关内容。

1. 阅读 [ SDK 集成指南 ](./INTEGRATION-GUIDE.md) 并打开 [ ANF Vulkan 示例 ](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/tree/main/samples/anf)。
2. 为 `arm64-v8a` 打包 `libanf.so`，在分发二进制文件时须包含 [`LICENSE.txt`](./LICENSE.txt)，并将公共标头添加到原生构建中。
3. 在创建 Vulkan 实例和设备之前查询技术要求。
4. 使用所需的扩展和队列功能创建 Vulkan，向 ANF 注册 Vulkan 句柄，并验证所选物理设备上的技术支持。
5. 根据 ANF 返回的要求创建技术及其资源。
6. 在技术渲染契约定义的时间点，从渲染器调度 SR 或 FG。
7. 在 SR 启动期间，使用 [ 调试覆盖层 ](./DEBUG-OVERLAY.md) 在调整图像质量之前验证输入。

集成指南是关于 API 使用、资源需求和同步的技术参考。该示例提供了一个可运行的实现以供对比。

## 调试 SR 集成

ANF 调试覆盖层可视化了 SR 使用的输入和时间行为。它可以直接在 SR 输出中显示输入颜色、深度、运动矢量、抖动覆盖率、翘曲预测和再投射误差。

使用 [ 调试覆盖层指南 ](./DEBUG-OVERLAY.md) 进行设置、模式选择、预期结果和伪影诊断。

## 贡献

使用 [ CONTRIBUTING.md ](./CONTRIBUTING.md) 获取问题报告、Pull Request、签核要求和文档约定。参与本存储库受 [ 行为准则 ](./CODE_OF_CONDUCT.md) 管辖。

## 许可

[`public/`](./public/) 的公共头文件根据 [ BSD 3-Clause 许可证 ](./LICENSE-BSD-3-Clause.txt) 提供。

[`lib/arm64-v8a/libanf.so`](./lib/arm64-v8a/libanf.so) 的预编译 SDK 二进制文件在 [QTI No-Login Binary License](./LICENSE.txt) 下分发。
