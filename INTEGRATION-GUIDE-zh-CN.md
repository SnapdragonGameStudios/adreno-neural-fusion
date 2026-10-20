# Adreno™ Neural Fusion (ANF) SDK 集成指南

| [English](./INTEGRATION-GUIDE.md) | [简体中文](./INTEGRATION-GUIDE-zh-CN.md) |
|---|---|

本指南介绍 ANF 与 Android Vulkan 渲染器的集成。

公共 SDK 支持两种时域技法：

- 超分辨率 （SR），通过低分辨率抖动渲染重建更高分辨率的场景图像。
- 帧生成 （FG），在渲染帧之间生成内插场景帧。

## 目录

- [渲染契约](#rendering-contract)
- [Android 项目配置](#android-project-configuration)
- [原生 API 集成](#native-api-integration)
- [技术方案创建](#technique-creation)
- [技术方案调度](#technique-dispatch)
- [时序状态与生命周期](#temporal-state-and-lifecycle)
- [输入验证](#input-validation)
- [调试可视化覆盖](#debug-visualization-overlay)
- [故障排查](#troubleshooting)
- [发布检查清单](#release-checklist)
- [相关资源](#related-resources)

## 渲染契约

ANF 是一个时域系统。它结合了当前帧的场景数据和内部历史记录。资源内容和帧间连续性与 Vulkan 格式和使用标志同样重要。

### 超分辨率资源流

以 SR 输入分辨率并带有投影抖动来渲染 3D 场景。在场景输入准备就绪后调度 SR。在 SR 之后合成 UI，并根据渲染器的管线安排其他输出分辨率的后处理。

| 资源 | 分辨率 | 抖动状态 | 用途 |
|---|---:|---|---|
| 输入颜色 | SR 输入尺寸 | 随场景抖动 | 当前低分辨率场景颜色 |
| 深度 | SR 输入尺寸 | 随场景抖动 | 当前场景深度 |
| 运动矢量 | SR 输入尺寸 | 排除投影抖动 | 帧之间的像素对应关系，包括摄像机和物体运动 |
| 抖动偏移量 | 子像素值 | 当前帧 | 应用于投影矩阵的偏移量 |
| 输出颜色 | SR 输出尺寸 | 不适用 | 重建的高分辨率场景颜色 |

`AnfSRCreateInfo` 声明单独的输入和输出尺寸。应用程序在创建和描述调度资源时，必须使用相同的尺寸。

将 UI、文本、准星以及其他屏幕空间叠加层从低分辨率时序输入中排除。在SR完成后对其进行合成，保证画面锐利且不进入时序历史。

### 帧生成资源流

FG 接受渲染帧的场景数据，并生成内插场景帧。 `AnfFGCreateInfo` 声明一个适用于输入和输出资源的尺寸。

| 资源 | 分辨率 | 抖动状态 | 用途 |
|---|---:|---|---|
| 输入颜色 | 声明的 FG 尺寸 | 无抖动 | 当前渲染的场景颜色 |
| 深度 | 声明的 FG 尺寸 | 无抖动 | 当前场景深度 |
| 运动矢量 | 声明的 FG 尺寸 | 无抖动 | 帧之间的像素对应关系，包括摄像机和物体运动 |
| 输出颜色 | 声明的 FG 尺寸 | 不适用 | 插值场景帧 |

将 FG 输出视为场景颜色。为显示的渲染帧和生成的帧分别单独渲染或合成 UI。这可以避免对文本、菜单和其他界面元素进行插值。

当 SR 输出馈送至 FG 输入时，FG 深度和运动矢量资源也必须使用 `AnfFGCreateInfo` 中声明的尺寸。当这些尺寸不同时，分辨率较低的 SR 输入尚无法直接传递。调度前准备匹配的 FG 资源。

### 运行时间要求

ANF 与分辨率无关。它不添加支持分辨率的固定列表，也不添加输入和输出尺寸的附加最小、最大或对齐限制。在创建技法时，应用程序会根据 Vulkan 设备的图像限制和可用内存选择尺寸。

对于 SR，输入和输出尺寸必须遵循所选质量模式定义的比率。`ANF_SR_QUALITY_MODE_PERFORMANCE` 在每个尺寸中使用 2.0x 比率。对于 FG，所有输入和输出资源必须使用 `AnfFGCreateInfo` 中声明的尺寸。

技法支持、Vulkan 扩展、队列功能、资源格式、使用标志和外部内存要求可能因设备和技法而异。在运行时查询它们，并根据返回的数据构建资源。

公共资源标签如下：

- `ANF_RESOURCE_LABEL_INPUT_COLOR`
- `ANF_RESOURCE_LABEL_DEPTH`
- `ANF_RESOURCE_LABEL_MOTION_VECTORS`
- `ANF_RESOURCE_LABEL_OUTPUT_COLOR`

不要假设每种技法都使用所有标签，或者某种格式适用于所有设备。`AnfTechniqueRequirements` 标识必需和可选的标签。`AnfResourceRequirements` 定义每个标签允许的资源类型和格式。

## Android 项目配置

`libanf.so` 是一个 Android ARM64 共享库。针对 `arm64-v8a` ABI 对其进行打包，并为同一 ABI 编译原生应用程序。

以下示例使用应用程序原生源码树下的独立 `anf` 目录。这是将预构建库及其头文件放在一起的常见方法，但 ANF 并不要求使用这些目录名称。只要 CMake 路径能正确解析，并且打包后的应用程序包含目标 ABI 的库，其他布局也能生效。

```text
app/
  src/
    main/
      cpp/
        CMakeLists.txt
        anf/
          include/
            ... ANF public headers ...
          lib/
            arm64-v8a/
              libanf.so
```

导入的 CMake 目标链接并打包共享库。确认最终的 APK 或 App Bundle 包含 `lib/arm64-v8a/libanf.so`。

所提供库采用16KB对齐的加载段。应用程序中的所有其他原生库还必须满足目标设备的页面大小要求。

### CMake 配置

声明导入的共享库目标并将其链接到应用程序的原生目标：

```cmake
if(NOT ANDROID_ABI STREQUAL "arm64-v8a")
    message(FATAL_ERROR "ANF requires the arm64-v8a ABI")
endif()

add_library(anf SHARED IMPORTED GLOBAL)

set_target_properties(anf PROPERTIES
    IMPORTED_LOCATION
        "${CMAKE_CURRENT_LIST_DIR}/anf/lib/${ANDROID_ABI}/libanf.so"
    INTERFACE_INCLUDE_DIRECTORIES
        "${CMAKE_CURRENT_LIST_DIR}/anf/include"
)

target_link_libraries(your_native_target PRIVATE anf)
```

包括通用 API、Vulkan 类型和应用程序所用技法的标头：

```cpp
#include <anf.h>
#include <anf_types_vk.h>
#include <anf_sr.h>  // When using Super Resolution
#include <anf_fg.h>  // When using Frame Generation
```

## 本机 API 集成

将每个 ANF 结构归零初始化并设置其匹配 `header.type`。仅将 `header.pNext` 用于为该调用定义的扩展结构。

### 1. 函数表

`GetAnfFunctions` 填充公共功能表。使用返回的指针，而不是直接调用单个 API 声明。

```cpp
AnfFunctions anf = {};
GetAnfFunctions(&anf);
```

应用程序必须在使用前验证每个必需的功能指针。

`QueryAnfVersion` 是可选的，仅用于诊断。

### 2. 实例创建

公共客户端 API 是 Vulkan。在启动期间启用日志记录，以便在 Logcat 中可见无效参数和不受支持的路径。

```cpp
static void AnfLogCallback(AnfLogLevel level, const char* message)
{
    // Route the message to the engine logger or Android log.
}

AnfInstanceCreateInfo instanceInfo = {};
instanceInfo.header.type           = ANF_STYPE_INSTANCE_CREATE_INFO;
instanceInfo.clientApi             = ANF_CLIENT_API_VULKAN;
instanceInfo.logLevel              = ANF_LOG_LEVEL_WARNING;
instanceInfo.logMessageCallback    = AnfLogCallback;

AnfInstance anfInstance = {};
AnfResult result = anf.CreateInstance(&instanceInfo, &anfInstance);
if (result != ANF_RESULT_SUCCESS)
{
    // Stop ANF initialization and use the application's fallback path.
}
```

对于正式发布版本，请选择与应用程序的日志记录策略匹配的日志级别。`ANF_LOG_LEVEL_NONE` 禁用 ANF 消息。

### 3. 技法和 Vulkan 要求

选择 `ANF_TECHNIQUE_ID_SR_TEMPORAL` 或 `ANF_TECHNIQUE_ID_FG_TEMPORAL`。在创建 Vulkan 实例和逻辑设备之前进行调用 `QueryTechniqueRequirements`。查询应用程序在同一个 Vulkan 设备生命周期内可能启用的所有技法。启用它们所需扩展的并集，并选择满足其队列标志的队列族。

```cpp
const AnfTechniqueRequirements* techniqueRequirements = nullptr;

result = anf.QueryTechniqueRequirements(
    anfInstance,
    techniqueId,
    &techniqueRequirements);

if ((result != ANF_RESULT_SUCCESS) ||
    (techniqueRequirements == nullptr) ||
    (techniqueRequirements->header.type != ANF_STYPE_TECHNIQUE_REQUIREMENTS))
{
    // Stop initialization before using techniqueRequirements.
    return false;
}

const AnfStructHeader* clientApiRequirements =
    techniqueRequirements->pClientApiRequirements;

if ((clientApiRequirements == nullptr) ||
    (clientApiRequirements->type !=
        ANF_STYPE_TECHNIQUE_VULKAN_API_REQUIREMENTS))
{
    // The returned client API requirements are not valid for Vulkan.
    return false;
}

const auto* vulkanRequirements =
    reinterpret_cast<const AnfTechniqueVulkanApiRequirements*>(
        clientApiRequirements);

// Enable every name in ppInstanceExtensions and ppDeviceExtensions.
// Select a queue family that supports all techniqueQueueFlags.
```

确认所有必需的实例扩展均可用。单独确认选定的物理设备支持所有必需的设备扩展。

当 `pTechniqueGroupRequirements` 非空时对其进行检查，并在对其进行强制类型转换前验证其 `type`。对于 SR，`ANF_STYPE_SR_REQUIREMENTS` 标识一个包含支持的质量模式位域的 `AnfSRRequirements` 结构体。

### 4. 硬件支持

创建 Vulkan 实例和设备后，将它们的句柄传递给 `IsTechniqueSupported`。

```cpp
AnfAdapterInfoVulkan adapterInfo = {};
adapterInfo.header.type          = ANF_STYPE_ADAPTER_INFO_VULKAN;
adapterInfo.instance             = vkInstance;
adapterInfo.physicalDevice       = vkPhysicalDevice;
adapterInfo.device               = vkDevice;

result = anf.IsTechniqueSupported(
    anfInstance,
    techniqueId,
    &adapterInfo.header);
```

仅 `ANF_RESULT_SUCCESS` 用于确认支持情况。通过选择非 ANF 渲染路径来处理 `ANF_RESULT_NOT_SUPPORTED`。将所有其他结果视为初始化错误，并停止 ANF 设置或使用回退路径。支持扩展的 Vulkan 设备不一定支持所有 ANF 技法。

### 5. Vulkan 设备与队列注册

在创建、调度或销毁技法之前调用 `SetInstanceClientAPIAdapterInfo`。一个 ANF 实例可以绑定到一个 Vulkan 设备。

立即调度还需要一个或多个队列。每个报告的队列能力位必须至少由一个提供的队列支持。

```cpp
AnfQueueInfoVulkan queueInfo = {};
queueInfo.header.type         = ANF_STYPE_QUEUE_INFO_VULKAN;
queueInfo.numQueues           = 1;
queueInfo.pQueues             = &graphicsQueue;
queueInfo.pQueueFamilyIndices = &graphicsQueueFamilyIndex;

queueInfo.header.pNext   = adapterInfo.header.pNext;
adapterInfo.header.pNext = &queueInfo.header;

result = anf.SetInstanceClientAPIAdapterInfo(
    anfInstance,
    &adapterInfo.header);

if (result != ANF_RESULT_SUCCESS)
{
    // Do not create the technique.
}
```

保持 Vulkan 实例、设备和提供的队列有效，直到所有技法都被销毁，并且 ANF 实例不再使用它们。

### 6. 资源要求

遍历由 `AnfTechniqueRequirements` 返回的所需标签。仅当应用程序计划提供这些标签时，才查询可选标签。

```cpp
for (uint32_t index = 0;
     index < techniqueRequirements->numRequiredResources;
     ++index)
{
    const AnfResourceLabel label =
        techniqueRequirements->pRequiredResources[index];

    const AnfResourceRequirements* resourceRequirements = nullptr;
    result = anf.QueryTechniqueResourceRequirements(
        anfInstance,
        techniqueId,
        label,
        &resourceRequirements);

    if (result != ANF_RESULT_SUCCESS)
    {
        // Abort technique setup.
        return false;
    }

    if ((resourceRequirements == nullptr) ||
        (resourceRequirements->header.type != ANF_STYPE_RESOURCE_REQUIREMENTS))
    {
        // Stop before using the returned requirements.
        return false;
    }

    const AnfStructHeader* clientApiRequirements =
        resourceRequirements->pClientApiRequirements;

    if ((clientApiRequirements == nullptr) ||
        (clientApiRequirements->type !=
            ANF_STYPE_RESOURCE_VULKAN_API_REQUIREMENTS))
    {
        // The returned resource requirements are not valid for Vulkan.
        return false;
    }

    const auto* vulkanResourceRequirements =
        reinterpret_cast<const AnfResourceVulkanApiRequirements*>(
            clientApiRequirements);

}
```

对于每个资源：

- 使用返回的 `resourceType`。
- 从 `pSupportedFormats` 中选择一种格式。
- 包含所有返回的 Vulkan 图像或缓冲区使用标志。
- 当 `externalMemoryType` 非零时，需遵从它。
- 使用技法创建信息中声明的尺寸。
- 保持资源存活，直到所有引用它的调度都已完成。

如果一个 Vulkan 图像被用于多个 ANF 技法或资源角色，请选择一种被所有适用要求支持的格式，并包含它们所需使用标志的并集。如果不存在通用格式，请使用单独的图像并在它们之间进行复制或转换。

请勿通过数值从 Vulkan 格式推断 `AnfFormat`。在渲染器中使用显式的转换表。

## 技法创建

### 超分辨率

下面的示例创建了 SR 以便立即调度。

```cpp
AnfTechniqueVulkanCreateInfo vulkanCreateInfo = {};
vulkanCreateInfo.header.type = ANF_STYPE_TECHNIQUE_VULKAN_CREATE_INFO;

AnfSRCreateInfo srCreateInfo = {};
srCreateInfo.header.type       = ANF_STYPE_SR_CREATE_INFO;
srCreateInfo.inputSize.width   = inputWidth;
srCreateInfo.inputSize.height  = inputHeight;
srCreateInfo.outputSize.width  = outputWidth;
srCreateInfo.outputSize.height = outputHeight;

AnfTechniqueCreateInfo createInfo = {};
createInfo.header.type               = ANF_STYPE_TECHNIQUE_CREATE_INFO;
createInfo.techniqueId               = ANF_TECHNIQUE_ID_SR_TEMPORAL;
createInfo.flags                     =
    ANF_TECHNIQUE_CREATE_FLAG_DISPATCH_IMMEDIATE;
createInfo.maxInFlight               = framesInFlight;
createInfo.pClientApiCreateInfo      = &vulkanCreateInfo.header;
createInfo.pTechniqueGroupCreateInfo = &srCreateInfo.header;

AnfTechnique srTechnique = {};
result = anf.CreateTechnique(anfInstance, &createInfo, &srTechnique);
```

仅当 `CreateTechnique` 返回 `ANF_RESULT_SUCCESS` 时继续。

对于已记录的调度，请忽略 `ANF_TECHNIQUE_CREATE_FLAG_DISPATCH_IMMEDIATE` 并遵循 [ 已记录模式 ](#recorded-mode)。

#### SR 调度抖动

`AnfSRDispatch::jitterOffset` 是应用于当前帧的投影矩阵的子像素偏移量。每个组件必须保留在 `[-0.5, 0.5]` 中。

```cpp
AnfSRDispatch srDispatch = {};
srDispatch.header.type    = ANF_STYPE_SR_DISPATCH;
srDispatch.jitterOffset   = currentJitter;
```

运动矢量输入应描述场景与相机运动，不含投影抖动偏移量。在调试阶段使用调试叠加层确认该规则。

### 帧生成

FG 使用即时调度。其输入和输出资源具有相同的宽度和高度。

```cpp
AnfTechniqueVulkanCreateInfo vulkanCreateInfo = {};
vulkanCreateInfo.header.type = ANF_STYPE_TECHNIQUE_VULKAN_CREATE_INFO;

AnfFGCreateInfo fgCreateInfo = {};
fgCreateInfo.header.type      = ANF_STYPE_FG_CREATE_INFO;
fgCreateInfo.inputSize.width  = frameWidth;
fgCreateInfo.inputSize.height = frameHeight;

AnfTechniqueCreateInfo createInfo = {};
createInfo.header.type               = ANF_STYPE_TECHNIQUE_CREATE_INFO;
createInfo.techniqueId               = ANF_TECHNIQUE_ID_FG_TEMPORAL;
createInfo.flags                     =
    ANF_TECHNIQUE_CREATE_FLAG_DISPATCH_IMMEDIATE;
createInfo.maxInFlight               = framesInFlight;
createInfo.pClientApiCreateInfo      = &vulkanCreateInfo.header;
createInfo.pTechniqueGroupCreateInfo = &fgCreateInfo.header;

AnfTechnique fgTechnique = {};
result = anf.CreateTechnique(anfInstance, &createInfo, &fgTechnique);
```

仅当 `CreateTechnique` 返回 `ANF_RESULT_SUCCESS` 时继续。

## 技法调度

### 资源描述符

为提供给调度的每个资源创建一个 `AnfResourceParamDesc`。其标签、类型、格式、尺寸、Vulkan 图像布局和句柄必须描述调度点的实际资源。

```cpp
AnfResourceParamDesc inputColor = {};
inputColor.header.type                       = ANF_STYPE_RESOURCE_PARAM_DESC;
inputColor.resourceLabel                     = ANF_RESOURCE_LABEL_INPUT_COLOR;
inputColor.resourceDesc.header.type          = ANF_STYPE_RESOURCE_DESC;
inputColor.resourceDesc.type                 = ANF_RESOURCE_TYPE_TEXTURE2D;
inputColor.resourceDesc.format               = inputColorAnfFormat;
inputColor.resourceDesc.layout               =
    static_cast<uint32_t>(inputColorLayout);
inputColor.resourceDesc.textureDims.width    = inputWidth;
inputColor.resourceDesc.textureDims.height   = inputHeight;
inputColor.resourceDesc.textureDims.numArrayLayers = 1;
inputColor.resourceDesc.textureDims.mipCount = 1;
inputColor.resourceDesc.resource             =
    reinterpret_cast<AnfClientResource>(inputColorImage);
```

根据所选技法的要求，对深度、运动矢量和输出颜色重复此操作。使用该技法预期的描述符提供每个所需的资源标签。按 `AnfTechniqueRequirements::pRequiredResources` 顺序列出资源以简化诊断。

`resourceDesc.layout` 中的值是图像当前的 `VkImageLayout`。调度前，将各图像转换为其描述符传入的布局，并使输入资源的早期写入操作对 ANF 可见。在后续命令消耗输出前，同步 ANF 的输出写入。

### 立即模式

在立即模式下，ANF 将工作提交到由 `AnfQueueInfoVulkan` 提供的队列。将 `commandList` 设置为 `nullptr`，并通过 `AnfTechniqueDispatchImmediateInfo` 提供 Vulkan 信号量。

```cpp
AnfClientSyncObject waitObjects[] = {
    reinterpret_cast<AnfClientSyncObject>(anfReadySemaphore)
};

AnfClientSyncObject signalObjects[] = {
    reinterpret_cast<AnfClientSyncObject>(anfDoneSemaphore)
};

AnfTechniqueDispatchImmediateInfo immediateInfo = {};
immediateInfo.header.type          =
    ANF_STYPE_TECHNIQUE_DISPATCH_IMMEDIATE_INFO;
immediateInfo.pWaitSyncObjects     = waitObjects;
immediateInfo.numWaitSyncObjects   = 1;
immediateInfo.pSignalSyncObjects   = signalObjects;
immediateInfo.numSignalSyncObjects = 1;

AnfTechniqueDispatchInfo dispatchInfo = {};
dispatchInfo.header.type                 = ANF_STYPE_TECHNIQUE_DISPATCH_INFO;
dispatchInfo.header.pNext                = &immediateInfo.header;
dispatchInfo.commandList                 = nullptr;
dispatchInfo.pResources                  = resources.data();
dispatchInfo.numResources                =
    static_cast<uint32_t>(resources.size());
dispatchInfo.reset                       =
    shouldResetHistory ? ANF_TRUE : ANF_FALSE;
dispatchInfo.pTechniqueGroupDispatchInfo = &srDispatch.header;

result = anf.DispatchTechnique(srTechnique, &dispatchInfo);
```

仅当 `DispatchTechnique` 返回 `ANF_RESULT_SUCCESS` 时继续。

对于 FG，使用 FG 资源描述符填充 `dispatchInfo.pResources` 和 `dispatchInfo.numResources`，初始化特定技法的调度结构，并使用 FG 技法句柄：

```cpp
AnfFGDispatch fgDispatch = {};
fgDispatch.header.type   = ANF_STYPE_FG_DISPATCH;

dispatchInfo.pResources                  = fgResources.data();
dispatchInfo.numResources                =
    static_cast<uint32_t>(fgResources.size());
dispatchInfo.pTechniqueGroupDispatchInfo = &fgDispatch.header;
result = anf.DispatchTechnique(fgTechnique, &dispatchInfo);
```

仅当 FG 调度返回 `ANF_RESULT_SUCCESS` 时继续。

等待信号量必须在输入资源及其所需的转换完成后，才会发出信号。在消耗输出之前，先等待信号量发出信号。ANF 信号量不能取代应用程序自身提交中的图像内存依赖关系。

优先选择拥有图像的同一队列系列中的 ANF 队列。如果 ANF 和渲染器在即时模式下使用不同的队列系列，请为这些系列创建带有 `VK_SHARING_MODE_CONCURRENT` 的共享图像。

### 录制模式

录制调度允许 ANF 向应用程序拥有的 `VkCommandBuffer` 添加命令。

- 在没有 `ANF_TECHNIQUE_CREATE_FLAG_DISPATCH_IMMEDIATE` 的情况下创建技法。
- 使用主命令缓冲区。
- 在 ANF 调用之前开始命令缓冲区。
- 在渲染通道或动态渲染范围之外记录 ANF 调度。
- 从满足 `techniqueQueueFlags` 的队列系列分配命令缓冲区。
- 将 `commandList` 设置为 `VkCommandBuffer`，并将 `dispatchInfo.header.pNext` 留空。

按如下方式配置录制模式调度：

```cpp
dispatchInfo.header.pNext = nullptr;
dispatchInfo.commandList =
    reinterpret_cast<AnfClientCommandList>(commandBuffer);
```

调用后，将命令缓冲区绑定状态视为未定义。重新绑定管道、描述符集、顶点和索引缓冲区、动态状态以及后续命令所需的任何其他状态。

应用程序仍然负责命令缓冲区提交、队列间依赖、图像转换和资源生命周期。

### 飞行中调度限制

`AnfTechniqueCreateInfo::maxInFlight` 是指一个技法实例可能仍在执行的调度的最大数量。

应将其与应用程序 N 缓冲模型中可能重叠的 ANF 提交数量相匹配，而不是自动匹配交换链图像数量。在应用程序重用或销毁其输入、输出、同步对象或技法特定数据之前，调度必须完成。

## 时序状态和生命周期

### 历史重置

当先前的时序数据不再描述当前帧时，将 `AnfTechniqueDispatchInfo::reset` 设置为 `ANF_TRUE`。常见的重置事件包括：

- 摄像机剪切或瞬移
 - 加载新场景或关卡
 - 视野或投影发生大幅变化
 - 需要重建技法的分辨率变化
 - 替换或清除时序输入资源
 - 长时间暂停或跳帧序列后恢复
 - 在未接收到连续帧后启用 ANF

在正常连续渲染期间使用 `ANF_FALSE`。频繁重置会破坏时序累积，从而降低画质。

### 技法重建与销毁

技法创建信息定义了尺寸、调度模式和 `maxInFlight`。当这些值发生变化时，销毁并重建技法，然后在创建替换资源之前再次查询资源需求。

ANF 接收 Vulkan 句柄，而不是图像内容的副本。在调度完成之前，保持所引用的图像、内存、信号量、命令缓冲区和 Vulkan 对象有效。在关闭或重建之前，请排空未完成的 ANF 工作，并按依赖顺序销毁对象：

```cpp
anf.DestroyTechnique(anfInstance, technique);
anf.DestroyInstance(anfInstance);
```

## 输入验证

### 投影抖动

在剪辑空间中为投影矩阵添加抖动。将每个分量保持在 `[-0.5, 0.5]` 像素内，并对渲染和 `AnfSRDispatch::jitterOffset` 使用相同的逻辑偏移量。矩阵存储、剪辑空间约定、视口方向和 Y 轴反转因渲染器而异，因此请使用调试覆盖层验证两个轴。

使用 Halton 或 Sobol 等低差异序列来覆盖像素区域，避免聚集。

### 运动矢量

运动矢量必须覆盖受摄像机移动以及动画和移动对象影响的静态几何体。从当前和先前的裁剪坐标到 UV 位移的参考转换是：

```glsl
vec2 CalculateMotionVector(vec4 currentClip, vec4 previousClip)
{
    vec2 currentNdc  = currentClip.xy / currentClip.w;
    vec2 previousNdc = previousClip.xy / previousClip.w;
    return 0.5 * (currentNdc - previousNdc);
}
```

渲染器可能需要翻转 Y 轴或反向方向，以匹配其纹理坐标和 ANF 输入约定。使用仅摄像机运动、仅对象运动和消遮挡检查整个帧。从 SR 使用的运动矢量信号中移除投影抖动。

### 深度

提供与颜色和运动矢量输入相同的摄像机、投影、视口和场景渲染的深度。不要传入陈旧的深度缓冲区或从其他视图解析得出的深度缓冲区。

深度格式支持取决于技法和设备。查询深度资源需求。仅在返回的格式列表包含它时使用 `ANF_FORMAT_D24S8_UNORM`。对于组合式深度/模板资源，ANF 仅使用其中的深度部分。模板数据不是技法的输入。

### 颜色管线和合成

输入颜色应包含应用程序 ANF 管道预期阶段的 3D 场景。保持帧间的颜色空间、曝光状态、Alpha 处理和格式一致。

对于 SR，在重建后应用需要输出分辨率细节的操作。UI 合成通常紧随 SR 之后。如果在 ANF 之后运行色调映射、泛光、锐化、胶片颗粒或颜色分级，这些通道也会影响调试覆盖层。

对于 FG，对提供时域输入的渲染帧使用匹配的场景颜色处理。在生成后合成当前 UI。

## 调试可视化覆盖层

调试覆盖层是 SR 集成工具。该工具可可视化运动矢量、深度、抖动、资源有效性以及 ANF 使用的其他数据。

覆盖层设置需要：

- 将 `ANF_INSTANCE_CREATE_FLAG_ENABLE_DEBUG_OVERLAY` 添加到实例标志中。
- 使用覆盖层标志创建实例后，查询 SR 资源需求并使用返回的使用标志创建资源。
- 将零初始化的 `AnfDebugOverlayConfig` 与 `ANF_STYPE_DEBUG_OVERLAY_CONFIG` 链接到 `AnfSRDispatch::header.pNext`。
- 选择由 `AnfDebugOverlayMode` 定义的模式之一。
- 发送一次 `ANF_DEBUG_OVERLAY_MODE_NONE` 以停止绘制。

覆盖层设置会持续生效。仅在值发生变化时链接新配置。

同一时间，一个调试覆盖层只能归属于一种技法。在创建使用该覆盖层的替代技法之前，请先销毁现有技法。

覆盖层会被写入 SR 输出中。后续的后处理可能会改变其颜色或遮挡部分内容。有关模式描述、控制、验证步骤和伪影诊断，请使用 [ ANF 调试覆盖层指南 ](./DEBUG-OVERLAY.md)。

## 故障排除

### 初始化和调度失败

检查 ANF 日志并确认：

- 所选设备已通过 `IsTechniqueSupported`。
- 每个必需的 Vulkan 扩展已在实例或设备创建时启用。
- `SetInstanceClientAPIAdapterInfo` 接收到了活动的 Vulkan 句柄。
- 立即模式接收到了其队列族满足 `techniqueQueueFlags` 的队列。
- 每个结构都有正确的 `header.type`。
- 特定技法的创建结构与所选技法 ID 匹配。
- `maxInFlight`匹配应用程序的缓冲计划。
- 输入和输出尺寸对于所选技法和设备是有效的。
- 每个必需的资源标签都存在。
- 资源格式、类型、尺寸、布局与查询所得的要求及技法创建信息相匹配。
- 调度模式与技法创建标志匹配。
- `pTechniqueGroupDispatchInfo` 指向正确的 SR 或 FG 结构。
- 未完成调度任务数量不超过 `maxInFlight`。

### OpenCL 加载失败

部分 Android 系统镜像会将 OpenCL 作为由设备提供的原生库对外提供。如果 ANF 报告 OpenCL 加载错误，则在应用程序元素中声明库为可选：

```xml
<application>
    <uses-native-library
        android:name="libOpenCL.so"
        android:required="false" />
</application>
```

保留 `android:required="false"` 可防止因清单文件声明而将安装限制在声明支持该库的设备上。运行时 ANF 支持检查仍用于判定该技术方案是否可执行。

### 输出无效或损坏

检查资源句柄、布局、资源屏障和信号量顺序。验证输出图像是否具有返回的使用标志，以及消费者是否等待 ANF 完成。

对于 SR，确认输入的颜色和深度是用分派出的抖动渲染出来的。确认运动矢量不包含抖动。对于 FG，确认所有输入和输出都使用 `AnfFGCreateInfo` 中声明的尺寸。

### 时域伪影

检查运动矢量、深度、抖动和复位行为。常见原因包括摄像机移动期间静态几何体上缺失运动、运动矢量方向反转、Y 轴不匹配、运动矢量中包含抖动、深度过期，或摄像机切换后未能重置。

在调整无关的后处理之前，使用调试覆盖层隔离出错的输入。

### 图块内存失效

在记录的调度期间，ANF 可能会使用 [`VK_QCOM_tile_memory_heap`](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_QCOM_tile_memory_heap.html) 进行内部工作。调度之后，将绑定到该堆的内存中的应用程序数据视为未定义。请勿在记录的 ANF 执行期间将应用程序数据保留在图块内存中。

## 发布清单

在发布版本中启用 ANF 之前，请确认以下项目。

为 `arm64-v8a` 打包匹配的公共标头和 `libanf.so`。
- 每次重新分发 `libanf.so` 时，必须附带来自 [`LICENSE.txt`](./LICENSE.txt) 的 QTI 无需登录二进制许可证。
- 在 Vulkan 实例和设备创建之前查询预期技法。
- 启用所有返回的扩展，并使用带有所需标志的队列。
- 运行 `IsTechniqueSupported` 并保留非 ANF 的回退路径。
- 根据返回的格式、使用情况和内存要求创建每个资源。
-将 `maxInFlight` 设置为应用程序的 N 缓冲模型中重叠的 ANF 提交数量。
- 保持资源和同步对象处于活动状态，直到调度完成。
- 对于 SR，抖动颜色和深度，但不包括运动矢量中的投影抖动。
- 中断后重置历史记录。
- 在 ANF 访问前后，对每个资源进行同步和状态转换。

## 相关资源

- [ Adreno™ Neural Fusion SDK 存储库](https://github.com/SnapdragonGameStudios/adreno-neural-fusion)
- [Adreno™ Neural Fusion 调试覆盖层指南](./DEBUG-OVERLAY.md)
- [Adreno™ Neural Fusion Vulkan 示例](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/tree/main/samples/anf)
- [用于虚幻引擎的 Adreno™ Neural Fusion 插件](https://github.com/SnapdragonGameStudios/snapdragon-game-plugins-for-unreal-engine#adreno-neural-fusion)
- [用于 Unity 的 Adreno™ Neural Fusion 插件](https://github.com/SnapdragonGameStudios/com.qualcomm.snapdragon.adreno.neural.fusion)
