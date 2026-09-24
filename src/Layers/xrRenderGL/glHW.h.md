# src/Layers/xrRenderGL/glHW.h

> Declares the device object: context creation, capability record, frame bracket and present.

**Needs** — [`glHW.cpp`](glHW.cpp.md) · [`xrRender/HWCaps.h`](../xrRender/HWCaps.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`glHW.cpp`](glHW.cpp.md) · [`glHWCaps.cpp`](glHWCaps.cpp.md) · [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md) · [`glr_screenshot.cpp`](glr_screenshot.cpp.md) · [`gl_rendertarget_phase_flip.cpp`](../xrRenderPC_GL/gl_rendertarget_phase_flip.cpp.md) · [`gl_rendertarget_u_set_rt.cpp`](../xrRenderPC_GL/gl_rendertarget_u_set_rt.cpp.md) · [`r2_test_hw.cpp`](../xrRenderPC_GL/r2_test_hw.cpp.md) · [`rgl_shaders.cpp`](../xrRenderPC_GL/rgl_shaders.cpp.md) · [`stdafx.h`](../xrRenderPC_GL/stdafx.h.md)
**Tier floor** — T1: it owns a foreign context handle whose current-thread binding is process state.

## Purpose

Declares the single device object, implemented in [`glHW.cpp`](glHW.cpp.md). One instance is global and owns the real window's context; further instances exist only for the capability probe in [`r2_test_hw.cpp`](../xrRenderPC_GL/r2_test_hw.cpp.md), and the object distinguishes the two so only the global one subscribes to application focus events.

Exported units:

- `create_device(window)` / `destroy_device` — bind a context to the window supplied by the windowing seam, load entry points, read the adapter strings, build the offscreen framebuffer.
- `set_primary_attributes(window_flags)` — what the windowing seam must be told *before* the window exists.
- `reset` — rebuild the framebuffer and re-read the vertical-sync preference after a mode change.
- `current_context` / `make_context_current(context)` — the render thread's claim on the context.
- `begin_scene` / `end_scene` / `present` — the frame bracket and the blit-and-swap.
- `surface_size`, `device_state` — queries the frame loop polls.
- `begin_debug_group(name)` / `end_debug_group` — nestable markers for a GPU capture tool.
- `on_app_activate` / `on_app_deactivate` — focus handling for exclusive display modes.
- `caps` — the capability record filled by [`glHWCaps.cpp`](glHWCaps.cpp.md).
- `back_buffer_count`, `current_back_buffer`, `framebuffer`, `adapter_name`, `api_version_string`, `shading_version_string`, `compute_shaders_supported` — read by the shader cache and the render-target code.
