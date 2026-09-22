# Contributing to Adreno™ Neural Fusion

Thank you for contributing to Adreno™ Neural Fusion (ANF). This repository contains the Android runtime library, public API headers, integration documentation, and links to the supported engine plugins and Vulkan sample.

Read the [code of conduct](./CODE_OF_CONDUCT.md) before participating. Contributions to the public headers are licensed under the [BSD 3-Clause License](./LICENSE-BSD-3-Clause.txt).

## Choose the right repository

Open the contribution where the affected code or documentation lives.

| Contribution | Repository |
|---|---|
| ANF SDK headers, native integration documentation, or this README | [Adreno™ Neural Fusion](https://github.com/SnapdragonGameStudios/adreno-neural-fusion) |
| Vulkan sample code | [Adreno™ GPU Vulkan Code Sample Framework](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/tree/main/samples/anf) |
| ANF plugin for Unreal Engine | [Snapdragon™ Game Plugins for Unreal Engine](https://github.com/SnapdragonGameStudios/snapdragon-game-plugins-for-unreal-engine#adreno-neural-fusion) |
| Unity plugin | [ANF plugin for Unity](https://github.com/SnapdragonGameStudios/com.qualcomm.snapdragon.adreno.neural.fusion) |

The ANF runtime is distributed as `lib/arm64-v8a/libanf.so` under the [QTI No-Login Binary License](./LICENSE.txt). Its implementation source is not part of this repository, so pull requests cannot change the runtime or its license.

## Report an issue

Search the [existing issues](https://github.com/SnapdragonGameStudios/adreno-neural-fusion/issues) before opening a new one.

For an integration or rendering issue, include:

- The ANF SDK version.
- The Snapdragon™ device platform.
- The Android version, GPU driver version, and engine version when applicable.
- The technique in use, SR or FG.
- Whether the application uses immediate or recorded dispatch.
- The input and output dimensions.
- A minimal sequence that reproduces the problem.
- Relevant ANF logs and Vulkan validation messages.
- Screenshots or captures that do not contain confidential project material.

For image-quality issues, describe the camera movement, scene content, and affected objects. State whether the artifact appears in input color, depth, motion vectors, jitter, or the final output. For SR issues, use the [debug overlay guide](./DEBUG-OVERLAY.md) to narrow down the input before filing the report.

Do not include private source code, unreleased game assets, credentials, or other sensitive information in a public issue.

## Propose a change

Open an issue before starting a large change, an API proposal, or a rewrite that affects several documents. This gives maintainers a chance to confirm the scope and avoid duplicate work.

Small documentation corrections and broken-link fixes can go directly to a pull request.

## Submit a pull request

1. Fork the [Adreno™ Neural Fusion repository](https://github.com/SnapdragonGameStudios/adreno-neural-fusion/fork).
2. Clone your fork.

   ```bash
   git clone https://github.com/<username>/adreno-neural-fusion.git
   cd adreno-neural-fusion
   ```

3. Add the upstream repository.

   ```bash
   git remote add upstream https://github.com/SnapdragonGameStudios/adreno-neural-fusion.git
   ```

4. Create a branch from the latest `main`.

   ```bash
   git fetch upstream
   git switch -c <branch-name> upstream/main
   ```

5. Make a focused change. Keep unrelated fixes in separate pull requests.
6. Validate the affected documentation, links, headers, or integration instructions.
7. Commit with a [Developer Certificate of Origin](https://developercertificate.org/) sign-off.

   ```bash
   git commit -s -m "Describe the change"
   ```

8. Push the branch and open a pull request against `main`.

   ```bash
   git push -u origin <branch-name>
   ```

   Open the pull request from the [repository pull-request page](https://github.com/SnapdragonGameStudios/adreno-neural-fusion/pulls).

## Pull-request expectations

A pull request should:

- Explain the problem and the proposed change.
- Link the related issue when one exists.
- Keep API names, enum values, resource labels, and file paths consistent with the public headers.
- Update nearby documentation when behavior or requirements change.
- Preserve working relative links between `README.md`, `INTEGRATION-GUIDE.md`, and `DEBUG-OVERLAY.md`.
- Avoid committing proprietary assets, generated build output, or unrelated binary files.
- Include the contributor sign-off on every commit.

For public API changes, describe the source and binary compatibility impact. Existing applications should not require changes unless the proposal explicitly introduces a versioned compatibility break.

## Documentation style

Use the existing guides as the style reference.

- Use sentence-case headings.
- Prefer short paragraphs and direct instructions.
- Format API types, functions, fields, enums, flags, paths, and commands with backticks.
- State requirements with `must`, `requires`, or `do not` when they are mandatory.
- Explain the visible failure or integration problem caused by an incorrect value.
- Avoid promotional filler and repeated explanations already covered by another guide.

The [SDK integration guide](./INTEGRATION-GUIDE.md) is the technical reference. The [debug overlay guide](./DEBUG-OVERLAY.md) is the SR diagnostic reference.
