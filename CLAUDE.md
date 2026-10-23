# DMS Depth: project brief for Claude Code

A native C++ After Effects 2026 SmartFX plug-in (Windows, NVIDIA first). It makes per-pixel depth maps with local ONNX Runtime inference (Depth Anything v2/v3 and a temporal video model). **Stability inside the host is the #1 requirement**: the plug-in must never crash, hang or leak inside After Effects.

The original brief and every rule derived from it are in `docs/STABILITY.md` (rule → code location → test → status). Read that file and `docs/ARCHITECTURE.md` before changing behavior.

## Layout

- `engine/` is a static library behind a C ABI (`engine/include/dms/engine.h`): `extern "C"`, status codes only, no exceptions cross it. It owns ONNX Runtime (loaded dynamically, never unloaded), the single inference worker thread, the fallback chain, the frame cache, the downloader, temporal windows, tiling and post-processing.
- `plugin/core/` is host-independent render code (params, flat sequence data, pixel conversion, sub-rect output). It is unit-tested on any OS.
- `plugin/ae/` is the thin After Effects glue (`PF_Cmd` dispatch, params UI, SmartFX, dialogs, GPU worlds). It is Windows-only and needs the AE SDK.
- `tools/depthcli/` is a CLI on the same engine. `tools/ae_sdk_mock/` is a compile-only SDK subset for Linux CI.
- `tests/unit/` holds the GoogleTest suite (99 tests). `tests/stress/` holds the CLI leak run plus the `aerender` stress harness.
- `scripts/` contains `setup-windows.ps1` (one-shot setup), `fetch-deps.ps1` (pinned, SHA-256-verified DLLs) and `install.ps1` (elevated).

## Build and test on Windows (this machine)

```powershell
powershell -ExecutionPolicy Bypass -File scripts\setup-windows.ps1 -Install   # everything; log: build\setup-log.txt
# or manually (generator from `cmake --help`, here "Visual Studio 18 2026"):
cmake -S . -B build\windows-release -G "Visual Studio 18 2026" -A x64 -DDMS_BUILD_PLUGIN=ON -DAE_SDK_ROOT=<sdk>
cmake --build build\windows-release --config Release
ctest --test-dir build\windows-release -C Release --output-on-failure
$env:DMS_DEPS_DIR="build\DMS_deps"; build\windows-release\bin\depthcli.exe --device
```

Developer machine facts:
- Hardware and OS: Windows 10 Pro, RTX 3080 10 GB, driver 591.44.
- Toolchain: VS Community 2026 18.11 with its bundled CMake 4.3.1, Python 3.11.
- Paths: the AE SDK 26.5 is on the Desktop and may move to `C:\SDKs`; After Effects 2026 is installed. Models live in `%LOCALAPPDATA%\DMS\models` and logs in `%LOCALAPPDATA%\DMS\logs\dms.log`. The installed plug-in is `C:\Program Files\Adobe\Adobe After Effects 2026\Support Files\Plug-ins\DMS\` (the `.aex` plus `DMS_deps\`).
- Last full run (2026-10-15): all steps passed. The `.aex` built against SDK 26.5 with `/W4 /WX`, 99/99 tests passed, and inference ran on CUDA. See `docs/VERIFICATION.md`.

## Rules for changes

- Warnings are errors (`/W4 /WX`, `-Werror` plus `-Wconversion`). All MSVC targets use the static CRT (`/MT`): never switch the `.aex` to `/MD`, because AE ships an older `msvcp140.dll`.
- Never change the parameter order or IDs in `plugin/core/params_model.h`. They are append-only, and the `ParamsModel.ParamOrderIsFrozen` test guards them.
- Plug-in layer: no raw `new` or `malloc`, RAII for every host resource, every handler inside `EffectMain`'s `try`/`catch`. No network, file dialogs or modal UI in render paths. Downloads happen only from `DO_DIALOG` or `USER_CHANGED_PARAM`.
- Engine: every exported function goes through `guard()`. Never unload ONNX Runtime or driver DLLs.
- After moving code referenced by `docs/STABILITY.md.in`, run `python tools/gen_stability_doc.py`; CI checks that the generated doc is current.
- Report only what was actually run, and update `docs/VERIFICATION.md` when something new is verified.

## Open items (next steps)

1. **AE checklist pending:** run the 14-step After Effects checklist at the end of `docs/BUILD_WINDOWS.md`. This is the first time the plug-in runs inside AE.
2. **Stress test:** run `tests\stress\aerender_stress.ps1` (4K × 500 frames per model × mapping × tiles).
3. **Model URLs needed:** Depth Anything v3 and the video model need download URLs plus SHA-256 hashes in `%LOCALAPPDATA%\DMS\models\models.json`. HuggingFace was unreachable from the cloud session.
4. **Untested backends and paths:** DirectML (needs a non-NVIDIA GPU), TensorRT (`-DDMS_ENABLE_TENSORRT=ON`), CUDA Memory Sharing, and the GPU render path (`plugin/ae/DMS_Depth_GPU.cpp`).
5. **Other risks:** see "Known risks" in `docs/STABILITY.md`: point-parameter downsample units, SmartFX output origin for cropped layers, and the `CUDA_MEMCPY2D` layout.

Development so far happened on branch `claude/ae-depth-map-plugin-5iubui`.
