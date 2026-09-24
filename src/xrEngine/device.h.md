# src/xrEngine/device.h

> Declares the device: the object that owns the window, the clocks, the camera, the frame loop and every per-frame callback list in the engine.

**Needs** — [`device.cpp`](device.cpp.md) · [`Render.h`](Render.h.md) · [`Stats.h`](Stats.h.md) · [`pure.h`](pure.h.md) · [`editor_base.h`](editor_base.h.md) · [`defines.h`](defines.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`pch.hpp`](../editors/xrWeatherEditor/pch.hpp.md) · [`engine_impl.cpp`](../editors/xrWeatherEngine/engine_impl.cpp.md) · [`ide.hpp`](../editors/xrWeatherEngine/ide.hpp.md) · [`pch.hpp`](../editors/xrWeatherEngine/pch.hpp.md) · [`CameraBase.h`](CameraBase.h.md) · [`CameraDefs.h`](CameraDefs.h.md) · [`CameraManager.cpp`](CameraManager.cpp.md) · [`CustomHUD.h`](CustomHUD.h.md) · [`Device_Initialize.cpp`](Device_Initialize.cpp.md) · [`Device_create.cpp`](Device_create.cpp.md) · [`Device_destroy.cpp`](Device_destroy.cpp.md) · [`Device_imgui.cpp`](Device_imgui.cpp.md) · [`Device_mode.cpp`](Device_mode.cpp.md) · [`Device_overdraw.cpp`](Device_overdraw.cpp.md) · _and 33 more_
**Tier floor** — T1: owns the window and the graphics device handles and publishes per-frame matrices read directly by device-facing code

## Purpose

Declares the surface implemented in [`device.cpp`](device.cpp.md) and, for the parts it does
not carry, in [`Device_create.cpp`](Device_create.cpp.md),
[`Device_destroy.cpp`](Device_destroy.cpp.md),
[`Device_Initialize.cpp`](Device_Initialize.cpp.md), [`Device_mode.cpp`](Device_mode.cpp.md),
[`Device_imgui.cpp`](Device_imgui.cpp.md) and
[`Device_overdraw.cpp`](Device_overdraw.cpp.md).

There is exactly one device. It is the spine of the program: every other subsystem either
registers a callback with it or reads the clock and camera it publishes.

Exported units:

- `CRenderDevice` — the device itself. Its state and the decisions behind each method are
  in [`device.cpp`](device.cpp.md).
- The callback registries: `seqRender`, `seqFrame`, `seqFrameMT`, `seqParallel`,
  `seqAppActivate`, `seqAppDeactivate`, `seqAppEnd`, `seqDeviceReset`, `seqUIReset`.
- The frame loop: `Run`, `ProcessFrame`, `BeforeFrame`, `FrameMove`, `DoRender`,
  `RenderBegin`, `Clear`, `RenderEnd`, `Shutdown`.
- Lifecycle: `Create`, `Destroy`, `Initialize`, `Reset`, `InitializeImGui`, `DestroyImGui`.
- Window and display: `ProcessEvent`, `OnWindowActivate`, `UpdateWindowProps`,
  `UpdateWindowRects`, `SelectResolution`, `FillVideoModes`, `CleanupVideoModes`,
  `SetWindowDraggable`.
- Time: `Pause`, `Paused`, `time_factor`, `TimerAsync`, `TimerAsync_MMT`, `GetTimerGlobal`.
- Camera: `OnCameraUpdated`, `SetNearer`.
- Precache: `PreCache`.
- `WaitEvent` — wait on a worker's completion without starving the platform's event queue.
- `CLoadScreenRenderer` — the frame and render hooks that keep the loading screen alive
  while no level is running.
- `CDeviceResetNotifier`, `CUIResetNotifier` — join the corresponding registry for the
  holder's lifetime.
- `g_loading_events` — the queue of load steps drained one per frame before any frame work.
- `load_screen_renderer`, `Device`, `g_bBenchmark` — the process-wide instances.

Two constants declared here are read by the renderer: the world's near plane (0.2) and the
first-person overlay's much closer one (0.05), which is why the overlay is rendered as a
separate pass with its own projection rather than drawn into the world.
