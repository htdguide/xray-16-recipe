# src/Layers/xrRenderDX11/dx11HW.h

> Declares the device object: bring-up, teardown, presentation, format probing and the device-state query.

**Needs** — [`dx11HW.cpp`](dx11HW.cpp.md) · [`xrRender/HWCaps.h`](../xrRender/HWCaps.h.md) · [`xrRender/stats_manager.h`](../xrRender/stats_manager.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`dx11SamplerStateCache.cpp`](StateManager/dx11SamplerStateCache.cpp.md) · [`dx11ShaderResourceStateCache.cpp`](StateManager/dx11ShaderResourceStateCache.cpp.md) · [`dx11StateCache.cpp`](StateManager/dx11StateCache.cpp.md) · [`dx11StateManager.cpp`](StateManager/dx11StateManager.cpp.md) · [`dx11BufferUtils.cpp`](dx11BufferUtils.cpp.md) · [`dx11HW.cpp`](dx11HW.cpp.md) · [`dx11HWCaps.cpp`](dx11HWCaps.cpp.md) · [`dx11R_Backend_Runtime.h`](dx11R_Backend_Runtime.h.md) · [`dx11SH_RT.cpp`](dx11SH_RT.cpp.md) · [`dx11Texture.cpp`](dx11Texture.cpp.md) · [`dx11r_screenshot.cpp`](dx11r_screenshot.cpp.md) · [`r2_test_hw.cpp`](../xrRenderPC_R4/r2_test_hw.cpp.md) · [`r4_shaders.cpp`](../xrRenderPC_R4/r4_shaders.cpp.md) · [`stdafx.h`](../xrRenderPC_R4/stdafx.h.md) · _and 1 more_
**Tier floor** — T1: it is a record of raw driver handles and a fixed-size context pool.

## Purpose

Declares the surface implemented in [`dx11HW.cpp`](dx11HW.cpp.md), plus the one piece of state every other file in the backend reads directly: the process-wide device instance. That global is the backend's own service locator — resource creation, state binding and the render-target set all reach it by name rather than being handed it.

## Exported units

- **the device record** — lifecycle (`CreateD3D`/`DestroyD3D`, `CreateDevice`/`DestroyDevice`, `Reset`), frame bracket (`BeginScene`, `EndScene`, `Present`), focus hooks, format probing (`CheckFormatSupport`, `SelectFormat` including an array form), `GetSurfaceSize`, `GetDeviceState`, `UsingFlipPresentationModel`, `SetPrimaryAttributes`.
- **`get_context(id)`** — returns one of the submission contexts by index, asserting the index is in range. The immediate context sits in the last slot, at index equal to the number of parallel contexts; that identity is a compile-time constant other files rely on.
- **the capability record, the chosen depth and target formats, the feature flags** (compute available, double precision, packed-difference instruction) — read by shader selection and by the fluid subsystem to decide whether a feature is available at all.
- **the resolved shader-compile entry point** — a function value, not a linked symbol; see the implementation twin for why.
- **the memory-statistics record and the profiler context** — optional instrumentation, compiled out in the tool builds.
- **the global instance** — one per process.

**Notes** — The whole class is compiled into more than one renderer module under a per-module namespace, which is how two backends that both call their device "the device" coexist in one executable. In a rebuild that is one type instantiated once per backend, not a naming trick.
