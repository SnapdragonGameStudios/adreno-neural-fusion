# Adreno™ Neural Fusion (ANF)

Mobile games are asked to do more with every generation. Players expect sharp images and smooth motion without giving up battery life, while developers have to fit the experience inside a mobile power budget.

Adreno™ Neural Fusion is Qualcomm's neural rendering SDK for mobile games. It gives game teams more room to improve visual quality and displayed frame rate while keeping power use under control. The application chooses its image dimensions, rendering order, and fallback behavior. ANF reports hardware support and Vulkan requirements at runtime so the renderer can configure the correct path for each device.

<p align="center">
  <img src="media/anf-hero.png" width="760" alt="Adreno™ Neural Fusion rendered sanctuary scene">
</p>

## Contents

- [Resources](#resources)
- [Techniques](#techniques)
- [Requirements](#requirements)
- [Package contents](#package-contents)
- [Getting started](#getting-started)
- [Debugging SR integration](#debugging-sr-integration)
- [Contributing](#contributing)
- [License](#license)

## Resources

| Resource | Use it for |
|---|---|
| [SDK integration guide](./INTEGRATION-GUIDE.md) | Native Android Vulkan setup, rendering requirements, technique creation, dispatch, synchronization, validation, and release checks |
| [Debug overlay guide](./DEBUG-OVERLAY.md) | Validating SR color, depth, motion vectors, jitter, warp prediction, and reprojection |
| [Snapdragon™ Profiler](https://www.qualcomm.com/developer/software/snapdragon-profiler) | Profiling and analyzing ANF-enabled applications on Snapdragon™ devices |
| [ANF Vulkan sample](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/tree/main/samples/anf) | A native Vulkan integration built on the Snapdragon™ Game Studios sample framework |
| [ANF plugin for Unreal Engine](https://github.com/SnapdragonGameStudios/snapdragon-game-plugins-for-unreal-engine#adreno-neural-fusion) | Integrating ANF into an Unreal Engine project |
| [ANF plugin for Unity](https://github.com/SnapdragonGameStudios/com.qualcomm.snapdragon.adreno.neural.fusion) | Integrating ANF into a Unity project |

## Techniques

### Super Resolution

SR reconstructs the scene at the output size selected by the application. The renderer supplies jittered scene color and depth, motion vectors without projection jitter, and the current jitter offset. Compose UI and other screen-space elements after SR so they remain sharp and stay out of temporal history.

See [Super Resolution resource flow](./INTEGRATION-GUIDE.md#super-resolution-resource-flow) for the rendering contract and required inputs.

### Frame Generation

FG uses consecutive rendered scene frames, depth, and motion vectors to create an intermediate scene frame. Render or composite UI separately for rendered and generated frames.

See [Frame Generation resource flow](./INTEGRATION-GUIDE.md#frame-generation-resource-flow) for input, output, and composition requirements.

<!-- IMAGE PLACEHOLDER
Simplified renderer-flow diagram showing:
- SR scene inputs to reconstructed scene output
- rendered scene frames to an FG-generated scene frame
- UI composed after the technique output
Use the same resource names and pass ordering as the SDK integration guide.
-->

## Requirements

The native SDK integration requires:

- An Android application built for `arm64-v8a`.
- Snapdragon™ 8 Elite Gen 6 and Higher (with broader support planned farther down our platform roadmap).
- A Vulkan renderer.
- A fallback path when the requested technique is unavailable.

The [SDK integration guide](./INTEGRATION-GUIDE.md) covers initialization, runtime support checks, and the complete rendering contract.

## Package contents

| Path | Contents |
|---|---|
| `lib/arm64-v8a/libanf.so` | Android ARM64 ANF runtime library |
| `public/anf.h` | Public API entry points and function table |
| `public/anf_types.h` | Common ANF types, structures, flags, and enums |
| `public/anf_types_vk.h` | Vulkan-specific ANF types |
| `public/anf_sr.h` | SR creation and dispatch structures |
| `public/anf_fg.h` | FG creation and dispatch structures |
| `DEBUG-OVERLAY.md` | SR debug overlay documentation |
| `INTEGRATION-GUIDE.md` | Native Vulkan integration and rendering contract |
| `LICENSE-BSD-3-Clause.txt` | BSD 3-Clause terms for the public headers |
| `LICENSE.txt` | QTI No-Login Binary License terms for `libanf.so` |

## Getting started

ANF can be integrated directly into a native Vulkan renderer or through the Unreal Engine and Unity plugins. For an engine integration, follow the instructions in the plugin repositories listed under [Resources](#resources). The steps below cover the native Vulkan SDK.

1. Read the [SDK integration guide](./INTEGRATION-GUIDE.md) and open the [ANF Vulkan sample](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/tree/main/samples/anf).
2. Package `libanf.so` for `arm64-v8a`, include [`LICENSE.txt`](./LICENSE.txt) with any redistribution of the binary, and add the public headers to the native build.
3. Query technique requirements before creating the Vulkan instance and device.
4. Create Vulkan with the required extensions and queue capabilities, register the Vulkan handles with ANF, and verify technique support on the selected physical device.
5. Create the technique and its resources from the requirements returned by ANF.
6. Dispatch SR or FG from the renderer at the point defined by the technique's rendering contract.
7. During SR bring-up, use the [debug overlay](./DEBUG-OVERLAY.md) to validate the inputs before tuning image quality.

The integration guide is the technical reference for API use, resource requirements, and synchronization. The sample provides a working implementation to compare against.

## Debugging SR integration

The ANF debug overlay visualizes the inputs and temporal behavior used by SR. It can display input color, depth, motion vectors, jitter coverage, warp prediction, and reprojection error directly in the SR output.

Use the [debug overlay guide](./DEBUG-OVERLAY.md) for setup, mode selection, expected results, and artifact diagnosis.

## Contributing

Use [CONTRIBUTING.md](./CONTRIBUTING.md) for issue reports, pull requests, sign-off requirements, and documentation conventions. Participation in this repository is governed by the [code of conduct](./CODE_OF_CONDUCT.md).

## License

The public header files in [`public/`](./public/) are available under the [BSD 3-Clause License](./LICENSE-BSD-3-Clause.txt).

The precompiled SDK binary at [`lib/arm64-v8a/libanf.so`](./lib/arm64-v8a/libanf.so) is distributed under the [QTI No-Login Binary License](./LICENSE.txt).
