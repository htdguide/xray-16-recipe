# src/xrEngine/Device_destroy.cpp

> Tears the graphics device down, and — separately — rebuilds it in place when the display mode changes or the device is lost.

**Needs** — [`device.h`](device.h.md) · [`Render.h`](Render.h.md) · [`xr_input.h`](xr_input.h.md) · [`pure.h`](pure.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: graphics resource lifetime and an ordered release.

## Purpose

Two operations that look similar and are not. `destroy` ends the device for good and empties every registration list, after which nothing may be rendered. `reset` rebuilds the device underneath a running game — a resolution change, a windowed/fullscreen switch, or a device lost by the driver — and everything that holds a graphics resource must be told, in the right order, so it can release and rebuild.

Device-lost recovery is the requirement that shapes this file. A rebuild targeting an API where devices are never lost still needs `reset`, because resolution changes go through the same path.

## `destroy`

**Contract** — Idempotent; returns immediately if the device is not ready. Lowers the ready flag first so nothing else attempts to render during the teardown, releases the statistics collector's resources, releases the renderer's own resources and then the renderer, compacts the allocator, clears every frame-sequence and reset-notification registration, and destroys the window last.

```text
FUNCTION destroy()
  IF NOT ready THEN RETURN
  ready = false                       # first: it is the guard everything else reads
  statistics.on_device_destroy()
  renderer.on_device_destroy(keep_textures = false)
  allocator.compact()
  renderer.destroy()
  clear every registration list: render, app-activate, app-deactivate, app-end,
        frame, frame-parallel, device-reset, parallel-tasks
  release statistics
  destroy window
```

**Invariants** — Clearing the registration lists rather than expecting registrants to deregister is a deliberate asymmetry: at shutdown, registrants are being destroyed in an order the device does not control, and a registrant that outlives the clear will simply fail to deregister into an empty list, which is harmless. The reverse — a live list holding a destroyed registrant — is not.

**Notes** — The renderer is told to release its resources and *then* told to destroy itself, in two calls. The split exists because the first call also serves device reset, where the renderer survives. The flag on the first call says whether textures may be kept resident across the release; at shutdown they may not.

## `reset`

**Contract** — Rebuilds the graphics device in place for a new window size or mode, without unloading the level. Releases input capture for the duration, notifies the debug overlay on both sides of the rebuild, applies the new window properties, resets the backend, restores default transform state, optionally re-warms the device by rendering a short burst of throwaway frames, then fires the device-reset notification chain and — only if the resolution actually changed — the UI-reset chain. Blocking; the elapsed time is logged because this is a user-visible stall.

```text
FUNCTION reset(precache = true)
  width_before, height_before = current resolution
  release input capture
  overlay.on_reset_begin()
  update_window_properties()
  renderer.reset(window, width, height, half_width, half_height)
  overlay.on_reset_end()
  update_window_properties()          # again — see Notes
  reset_transform_state()
  IF precache THEN run 20 warm-up frames without waiting for the user
  log elapsed milliseconds
  allocator.compact()

  device_reset_chain.notify()                       # everyone rebuilds graphics resources
  IF resolution changed THEN ui_reset_chain.notify() # only then does the UI re-lay out
  IF NOT dedicated_server THEN re-acquire input capture
```

**Notes** — Window properties are applied twice, before and after the backend reset. The original marks the second as a workaround: resetting the backend can change the window's actual size behind the engine's back — an exclusive-fullscreen mode switch may land on a mode the driver chose rather than the one requested — and the second pass re-reads the truth. A rebuild should instead *read back* the achieved mode after the reset rather than re-applying the requested one, which is what the second call accidentally accomplishes.

**Notes** — Two notification chains, fired conditionally on different things. Device reset always fires: every graphics resource is gone and must be rebuilt regardless of whether the size changed. UI reset fires only on a size change, because re-laying out every screen is expensive and a windowed-to-borderless switch at the same resolution does not need it.

**Notes** — Input capture is dropped for the whole operation and re-acquired at the end, and the dedicated server never re-acquires it. Holding capture across a fullscreen transition leaves the pointer confined to a window that no longer exists at that size.

**Notes** — The warm-up burst renders twenty frames that are never seen. Its purpose is to force the backend to re-upload and re-validate the resources the reset chain just rebuilt, so the first frame the player actually sees is not the one that pays for all of it. Twenty is enough to cover the device's own internal frame pipelining several times over; the exact number is a judgement call.

**Notes** — The allocator compaction inside reset is marked in the original as something that should be removed because it may mask a crash — it hides a use-after-free by returning freed memory to the system where it will fault loudly later instead of immediately. A rebuild should not include it.
