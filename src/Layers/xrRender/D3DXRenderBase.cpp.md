# src/Layers/xrRender/D3DXRenderBase.cpp

> The renderer's lifecycle and frame bracket: bring up a device, build the shared resources, open and close each frame across every context, and report the frame's cost.

**Needs** — [`D3DXRenderBase.h`](D3DXRenderBase.h.md) · [`D3DUtils.h`](D3DUtils.h.md) · [`dxUIRender.h`](dxUIRender.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`HWCaps.h`](HWCaps.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Profiler and GPU debugging](../../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging)
**Used by** — reached through its declarations in [`D3DXRenderBase.h`](D3DXRenderBase.h.md); callers name that, not this file.
**Tier floor** — T1: it brackets the device's scene, owns the resource manager's lifetime and releases device objects in a defined order.

## Purpose

The renderer's top-level plumbing. Nothing here decides how a frame *looks*; it decides when the device exists, what is built on top of it, in what order everything is torn down, and what the frame bracket does. Almost every method is three lines, and the value of the page is the **ordering**, which is scattered across those three-line bodies and written down nowhere in the original.

## Startup ordering

```text
FUNCTION create(window) -> surface size
  bring up the graphics device against the window
  read back the surface size and its half-extents      # cached; the UI projects with them
  construct the resource manager

FUNCTION obtain_required_window_flags(flags)
  # runs BEFORE the window exists: the backend states what kind of surface it needs
  # (which graphics API, whether it must be a high-DPI surface, and so on)

FUNCTION setup_states()
  update the capability record from the live device
  push initial render state into every context's command list

FUNCTION on_device_create(shader_name)
  create the shared dynamic vertex and index streams
  create the shared quad index buffer
  bring up every context's command list
  apply the gamma curve
  resource manager: open the material library, check it for compatibility
  the generation-specific create hook            # render targets, backend materials
  IF this process renders at all
    resolve four fixed materials: wire, selection, portal fade, and the fade geometry
    bring up the shape-drawing toolbox and the UI vertex sink
```

**Invariants**

- The window flags are negotiated **before the window is created**, which is why that method exists at all rather than the renderer configuring the window afterwards. A graphics API is chosen at window-creation time on most platforms and cannot be changed after.
- Capabilities are read **after** the device exists and **before** any material compiles, because the material compiler branches on them.
- The **material library is opened before any material is resolved**, and its compatibility check runs immediately after. The shipped library is version-tagged and a mismatch is refused, not tolerated.
- The dedicated-server configuration skips everything player-facing — the shape toolbox, the UI sink, the four fixed materials — and still brings up streams, contexts and resources. That is because the server links the same renderer and calls the same entry points; the seam that would let it link none of this does not exist. A rebuild should make the renderer genuinely absent on a server rather than present and inert.

## Teardown ordering

```text
FUNCTION on_device_destroy(keep_textures)
  IF this process renders
    release the UI geometry, the shape toolbox, and the four fixed materials
  the generation-specific destroy hook
  resource manager: release device objects, optionally keeping texture data
  every context's command list: release device objects
  release the quad index buffer
  destroy the index stream, then the vertex stream

FUNCTION destroy()
  destroy the resource manager
  destroy the graphics device
```

**Invariants**

- **Exactly the reverse of creation**, and it has to be: the materials reference textures, the textures reference the device, and the streams are referenced by geometry declarations that the materials hold. Releasing the device with a material still holding a texture is the classic leak this ordering prevents.
- `keep_textures` is the level-transition path: the textures a *new* level also needs should not be released and re-decoded. See the resource manager's store/destroy pair, which decides which those are.
- `destroy` is separate from `on_device_destroy` because the first is per device and the second is per process. A resolution change runs the second and not the first.

## `reset(window) -> new surface size`

**Contract** — rebuilds every device-dependent resource after a resolution change, a fullscreen toggle or a device loss. Blocks for as long as it takes. Returns the new surface size and its half-extents.

```text
FUNCTION reset(window)
  the generation-specific reset_begin hook        # release render targets
  compact the allocator                           # see note
  reset the graphics device
  re-read the surface size and half-extents
  resource manager: rebuild what the reset invalidated
  the generation-specific reset_end hook          # recreate render targets
```

**Notes** — Compacting the allocator in the middle of a device reset is opportunism: the reset is already a visible stall, so it is the one moment in the process's life when a heap compaction is free. It has nothing to do with the reset itself. A rebuild on a collecting runtime does the same thing for the same reason.

## The frame bracket

```text
FUNCTION begin()
  device: begin scene
  FOR EACH context's command list
    begin frame
    set cull mode clockwise, then counter-clockwise     # see note
  flush both dynamic streams                            # reclaim last frame's space
  IF overdraw visualization is on, begin it

FUNCTION clear()
  clear the depth buffer to far, stencil to zero
  IF the "clear back buffer" flag is set, clear the colour target to black

FUNCTION end()
  IF overdraw visualization is on, end it
  FOR EACH context's command list, end frame
  reset every context                                   # nothing survives the frame
  device: end scene
  device: present
```

**Invariants**

- **The cull mode is set twice, to two different values, deliberately.** The command list caches its state and skips redundant sets; setting the opposite value first forces the cache to register a real change, so the counter-clockwise setting is actually pushed to the device. It is a cache-priming hack around a write-through-on-change optimization, and it is exactly the kind of thing that survives as its problem: *after a device reset the state cache and the device disagree, and the frame must begin by making them agree.* A rebuild invalidates the cache instead.
- **The back buffer is cleared only on request.** The renderer normally covers every pixel, and clearing is a measurable cost at high resolution. The flag exists for debugging a frame that does not.
- Presenting is the last thing `end` does, and the contexts are reset *before* it. A context holding a reference to a resource the present might need is therefore not a concern, because command lists were already submitted.

## `set_cache_transform(view, projection)`

**Contract** — pushes the camera to every context's command list at once. Called when the camera changes, which for the main view is once per frame and for shadow passes is once per cascade.

## Resource passthroughs

**Contract** — deferred load enable/disable, deferred upload and unload, memory accounting, the store/destroy pair around a level change, and the memory dump. Every one is a one-line delegation to the resource manager, and they exist only because the engine reaches the renderer through an interface that does not expose the resource manager directly.

**Notes** — That is the honest reading: a dozen methods on the renderer interface exist purely to forward to an object the interface chose not to expose. A rebuild that exposes a resource handle on the interface deletes all of them.

## `supports_shader_yuv_conversion()`

**Contract** — reports whether video frames can be converted from their decoded plane format on the device rather than on the processor. Answers yes on any device with a programmable pixel stage, which is every supported device.

## `dump_statistics(font, alert)`

**Contract** — writes the developer overlay's render page and raises alerts past three thresholds. Called once per frame when the overlay is up; ends and restarts the statistics frame as a side effect.

**Invariants**

- The per-stage timings are printed both absolutely and as a percentage of the frame's render total, which is the only way the numbers are actionable.
- The draw-call breakdown is by **draw-stream bucket** — static, flora, flora level-of-detail, dynamic, dynamic-software-skinned, dynamic-instanced, and dynamic by bone-influence count (one through four), and details. Those buckets are the draw stream's own sort categories (see [`r__dsgraph_types.h`](r__dsgraph_types.h.md)), which makes this overlay a direct readout of how the scene sorted. A rebuild that changes the sort key changes this page.
- Three alert thresholds: half a million vertices, a thousand draw calls, a thousand detail objects. They are the shipped engine's own statement of what "too much" means at its target frame rate on hardware of the era, and they are worth keeping as a sanity check even when the absolute numbers are no longer alarming.

## The frame-capture integration

**Contract** — on a non-shipping build, the renderer looks for a GPU capture tool already loaded into the process, or loads it, and configures it: where captures go, which key triggers one, and several capture options.

**Notes** — It disables the tool's own crash handler because the engine installs its own and the two conflict. It also forces vertical sync off during capture and turns on several expensive validation options — this is a developer path and is compiled out of a shipping build entirely. Wholly optional in a rebuild; see the profiler seam.
