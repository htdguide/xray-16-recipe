# src/Layers/xrRenderGL/glHW.cpp

> Brings up the drawing context on a window the windowing seam created, and defines what "present a frame" means when every frame is drawn into an offscreen framebuffer.

**Needs** — [`glHW.h`](glHW.h.md) · [`glHWCaps.cpp`](glHWCaps.cpp.md) · [`CommonTypes.h`](CommonTypes.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glHW.h`](glHW.h.md)
**Tier floor** — T1: the context is current per OS thread, entry points are resolved by name at run time, and present is a driver-synchronized buffer swap.

## Purpose

This is where the graphics-device seam meets the windowing seam. It states three things a rebuild must reproduce exactly: what the window must have been created *with* before a context can be made (so the two seams have to agree before either is used), what the render thread must hold to issue any device call at all, and what "present" means for a renderer that never draws to the window's own buffer.

The last one is the surprise. The deferred renderer needs a depth-stencil target it can also sample, and multiple colour attachments bound at once; the window's own buffer offers neither. So the device creates one offscreen framebuffer at startup, every phase binds its attachments into *that*, and present is a full-surface copy from it to the window's buffer followed by a swap. A rebuild on an API where the swap-chain image is a normal texture can drop the copy; on this one it is unavoidable and costs one full-resolution blit per frame.

## State

```text
RECORD Device
  window                : handle from the windowing seam
  context               : handle, valid only on the thread that made it current
  framebuffer           : int          # the offscreen target every phase binds into
  back_buffer_count     : int          # always 1 here; the swap chain is the window's
  current_back_buffer   : int          # cycles modulo back_buffer_count each present
  caps                  : Capabilities # see glHWCaps.cpp
  adapter_name          : text         # renderer string, read once
  api_version           : text
  shading_version       : text
  compute_supported     : bool         # always false: the compute path is unimplemented
  is_global             : bool         # false for the throwaway probe instance

# Invariant: exactly one instance is the global one, and only that instance
# subscribes to application focus events and touches vertical sync. The probe
# instance in r2_test_hw creates and destroys a context on a hidden window and
# must not disturb the running device's settings.
# Invariant: no device call may be issued unless this thread holds the context.
```

## `set_primary_attributes(window_flags)`

**Contract** — states the context requirements the windowing seam must satisfy *before* it creates the window, because on this API the pixel format is chosen at window-creation time and cannot be changed afterwards. Adds the flag that marks the window as context-capable and requests: a core profile (no legacy fixed-function entry points), eight bits each of red, green, blue and alpha, double buffering, twenty-four bits of depth and eight of stencil, and version 4.1.

**Notes** — Two numbers here are load-bearing and neither is arbitrary. The depth-stencil request of 24+8 matches the format the backend nominates for its own depth targets, so the window's buffer and the offscreen targets agree. The version 4.1 request is the floor the shipped shader sources declare (see [`rgl_shaders.cpp`](../xrRenderPC_GL/rgl_shaders.cpp.md)), and it is also the ceiling one major desktop platform ever shipped — which is why this backend, not the Direct3D one, is the portable filling.

A command-line switch suppresses the version request entirely, leaving the windowing seam to hand back whatever it likes. That exists to diagnose a driver that refuses the explicit request; it is not a supported configuration.

## `create_device(window)`

**Contract** — makes the device usable. Creates the context on the given window, claims it on the calling thread, resolves every device entry point by name, reads the adapter's identifying strings, and builds the offscreen framebuffer. Blocks while the driver compiles its state. On any failure it logs and returns with the device unusable rather than aborting — the caller (the renderer-selection path) treats that as "this backend is not available" and tries the next candidate, which is the seam's fallback contract.

```text
FUNCTION create_device(window) -> ()
  self.window := window

  # Pin the surface format before the context exists.
  mode := windowing.display_mode_of(window)
  mode.pixel_format := 8-bit RGBA
  windowing.set_display_mode(window, mode)

  caps.target_format := 8-bit BGRA
  caps.depth_format  := 24-bit depth + 8-bit stencil

  self.context := windowing.create_context(window)
  IF self.context IS none THEN
      log("could not create drawing context"); RETURN

  IF make_context_current(primary) FAILS THEN
      log("could not make context current"); RETURN

  # Entry points can only be resolved once a context is current, and they are
  # resolved against THIS context's driver. A rebuild on a language with a
  # static binding still needs this step: the symbols live in the driver, not
  # in any library the program linked against.
  IF NOT load_entry_points(windowing.proc_address) THEN
      log("could not initialize entry point loader"); RETURN

  IF is_global THEN
      apply_vsync_preference()
      IF debug build AND driver offers a message callback THEN
          route driver diagnostics to the engine log, dropping notification severity

  adapter_name         := device.renderer_string
  api_version          := device.version_string
  shading_version      := device.shading_language_version_string
  log vendor, adapter, versions, and the vertex- and combined-texture-unit counts

  compute_supported := false      # the compute path is not implemented on this backend

  IF the device can create framebuffers THEN
      create_offscreen_framebuffer()
```

**Invariants** — after a successful return, the calling thread holds the context and the offscreen framebuffer is bound. The adapter and version strings are captured here and never re-read; the compiled-shader cache keys on them (see [`rgl_shaders.cpp`](../xrRenderPC_GL/rgl_shaders.cpp.md)), so a driver update invalidates the cache automatically.

**Notes** — The three identifying strings are the entire device-identity story on this backend. The other backend can ask for a vendor and device number; here there is only text, so the cache compares text. That is weaker but sufficient: a driver that changes its behaviour without changing its version string is a driver bug.

## `present`

**Contract** — publishes the frame. Copies the offscreen framebuffer's colour to the window's buffer at one-to-one scale with nearest sampling, then asks the windowing seam to swap. Advances the back-buffer counter modulo the buffer count. Blocks for as long as the swap interval demands.

```text
FUNCTION present() -> ()
  bind framebuffer AS read source
  bind window's own buffer AS draw destination
  copy colour, full surface to full surface, nearest sampling
  windowing.swap(window)
  current_back_buffer := (current_back_buffer + 1) MOD back_buffer_count
```

**Notes** — An earlier design drew a textured full-screen quad instead of copying, so a shader could do the final transfer; that path is retired because the copy is both simpler and faster, and because there is nothing left for the shader to do — tone mapping and post-processing already ran into the offscreen target. A rebuild should copy.

## `reset`

**Contract** — re-establishes the device after a resolution or display-mode change. Destroys and rebuilds the offscreen framebuffer (its attachments are sized to the old surface) and re-applies the vertical-sync preference, which some drivers drop across a mode change. Does not recreate the context: on this API a mode change does not lose it, which is why the device-lost path the interface demands is, here, a no-op — `device_state` always answers *normal*.

**Notes** — This is the one place where the seam's "device-lost path that rebuilds every resource" requirement is satisfied trivially rather than implemented. A rebuild targeting an API that *can* lose a device must implement it properly; a rebuild on an API that cannot should still route through the same entry point, because the callers (render targets, textures, buffers) all have reset hooks that assume it exists.

## `make_context_current(context)` · `current_context`

**Contract** — claims or releases the context on the calling thread, and reports whether this thread currently holds it. Two states only: *none* and *primary*. Any other value is a programming error.

**Notes** — This is the whole of this backend's multi-context story, and it is worth being blunt about it: there is one context, it belongs to one thread at a time, and the deferred renderer's several command lists all drain through it. The interface exposes a context identifier per command list because the Direct3D 11 backend can genuinely record on several threads; here every identifier resolves to the same context. A rebuild on a modern API should honour the interface and record in parallel, and will find that the engine's render graph already supports it.

## `apply_vsync_preference` (private)

**Contract** — asks the windowing seam for adaptive synchronization first, and falls back to plain synchronization if the driver refuses; with synchronization off, requests no wait at all. Adaptive is preferred because it tears rather than halving the frame rate when a frame overruns, which matters for a loop that targets a fixed sixty frames per second with a long tail.

## `create_offscreen_framebuffer` (private)

**Contract** — creates the one framebuffer every render phase binds its attachments into, binds it, and records a buffer count of one. Nothing is attached here; attachments are bound per phase by [`u_setrt`](../xrRenderPC_GL/gl_rendertarget_u_set_rt.cpp.md).

## `begin_scene` · `end_scene`

**Contract** — empty. The frame bracket exists because the interface demands it and the other backend needs it; this API has no such notion.

## `on_app_activate` · `on_app_deactivate`

**Contract** — on regaining focus, restore the window. On losing focus, minimize it *only* when the window is fullscreen or borderless-fullscreen, so the player can reach other windows; a plain windowed session is left alone. Only the global instance subscribes.

## `begin_debug_group(name)` · `end_debug_group`

**Contract** — pushes and pops a named marker in the device's command stream so a capture tool can attribute work to a render phase. Silently does nothing when the driver lacks the facility. Every render phase brackets itself with these; they are the whole of the GPU-debugging seam on this backend.

## `surface_size`

**Contract** — reports the configured display width and height. Note that it answers from the *settings*, not from the window — the offscreen targets are sized from the same settings, so the two always agree, and asking the window would introduce a frame of disagreement during a resize.

## `device_state`

**Contract** — always answers *normal*. See `reset`.
