# src/xrEngine/Device_create.cpp

> Creates the graphics device behind the already-existing window, loads the material library, and declares the device ready.

**Needs** — [`device.h`](device.h.md) · [`Render.h`](Render.h.md) · [`Stats.h`](Stats.h.md) · [`Device_mode.cpp`](Device_mode.cpp.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it drives graphics-device creation and asks the allocator to compact.

## Purpose

The second half of bring-up, run after the window exists. It is idempotent by design: calling it when the device is already ready returns immediately, because several startup paths can reach it (a renderer change from the console, a first level load, the editor's own boot) and none of them can easily prove they are first.

## `create`

**Contract** — Creates the statistics collector, resolves the window to its final size and mode, creates the graphics device against the window, loads the material library, creates the debug-overlay renderer, and marks the device ready. Blocking, and slow — material library loading compiles or loads cached shader blobs for everything the game can draw. After it returns, the frame counter is zero and the first frame may run.

```text
FUNCTION create()
  IF already ready THEN RETURN

  statistics = new collector
  fov = 90 ; aspect = 1
  IF dedicated_server THEN force windowed mode
  update_window_properties()                 # resolves and applies the real size/mode
  renderer.create(window, width, height, half_width, half_height)
  allocator.compact()
  ready = true

  reset_transform_state()
  renderer.on_device_created(path of the material library, "$game_data$/shaders.xr")
  overlay_renderer = render_factory.create_overlay_renderer()
  overlay_renderer.on_device_created(overlay_context)
  statistics.on_device_created()
  frame_counter = 0
```

**Notes** — The ready flag is raised *before* the material library loads, not after. That is deliberate: loading materials issues graphics-device calls, and everything downstream of the device guards on the ready flag. Raising it late would make the material load fail its own guards.

**Notes** — The allocator is compacted immediately after device creation because the graphics backend has just made and released a large number of transient allocations while probing formats and capabilities, and the level load that follows wants one contiguous multi-hundred-megabyte region. Compaction here is cheap; failing to find that region later is fatal.

**Notes** — A dedicated server forces windowed mode rather than skipping device creation. The server still has a device and still runs the loop; it simply never draws a scene. A rebuild may genuinely skip the graphics device for a server, at the cost of splitting the loop.

## `reset_transform_state`

**Contract** — Returns the device's view and projection matrices to identity and the camera basis to the world axes, then asks the backend to restore its own default render state. Run at creation and after every device reset, so that the first frame after a mode change never draws with a matrix left over from the previous device.

**Notes** — The engine's depth convention is fixed here and assumed everywhere downstream: the near plane maps to 0 and the far plane to 1. Every projection matrix built anywhere in the engine must agree, and a backend whose native convention differs must convert rather than propagate.
