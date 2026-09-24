# src/Layers/xrRenderDX11/dx11SH_RT.cpp

> A render target: an allocation that is simultaneously something the frame writes into and something a later pass samples, with the views that make both possible and the rebuild path for device loss.

**Needs** — [`dx11TextureUtils.h`](dx11TextureUtils.h.md) · [`dx11HW.h`](dx11HW.h.md) · [`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [`xrRender/SH_RT.h`](../xrRender/SH_RT.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx113DFluidRenderer.cpp`](3DFluid/dx113DFluidRenderer.cpp.md) · [`r4_rendertarget.h`](../xrRenderPC_R4/r4_rendertarget.h.md) · [`r4_rendertarget_u_set_rt.cpp`](../xrRenderPC_R4/r4_rendertarget_u_set_rt.cpp.md)
**Tier floor** — T1: it allocates device memory in a specific layout and must be released and rebuilt in a defined order.

## Purpose

The deferred renderer's whole structure is "write into targets, then read those targets". This file is the target. What makes it more than an allocation is that **every target must be describable two ways at once** — as something the output stage writes and as something a shader samples — and for depth targets that is only possible if the allocation itself is declared without a fixed interpretation and the interpretations are attached afterwards, one per view.

The second job is the **device-loss rebuild**: a target knows everything needed to recreate itself, so losing the device costs only the contents.

## State

```text
RECORD RenderTarget
  name          : text
  width, height : int
  format        : enum      # the engine's vocabulary
  sample_count  : int       # 1 means no multisampling
  slice_count   : int       # >1 makes it an array, used for cascaded shadow maps
  flags         : set       # {derived-from-the-presentation-chain, writable-from-a-shader}
  surface       : handle    # the allocation
  colour_view   : handle    # present for colour targets
  depth_view_all: handle    # present for depth targets: the whole array
  depth_view_per_slice : list<handle>   # one per slice
  bound_depth_view : per-context handle # which of the above this context writes through
  unordered_view: handle    # present only when shader-writable was requested and granted
  texture       : Texture   # the sampling side, sharing the same allocation
  order         : int       # creation timestamp, used by the eviction policy
```

Invariants: a target with a depth format has depth views and no colour view; a colour target the reverse. `bound_depth_view` defaults to the whole-array view, and pointing it at one slice is how a cascade is rendered.

## `create`

**Contract** — allocates a target and every view it needs. Silently does nothing if already created. Refuses, by returning without allocating, when the requested size exceeds the device's maximum texture edge or when the device reports the format unusable for the requested role — the caller checks validity afterwards and degrades.

```text
FUNCTION create(name, w, h, format, samples, slices, flags)
  IF already allocated THEN RETURN
  REQUIRE device exists, name non-empty, w and h non-zero
  order = current high-resolution timestamp        # for the eviction policy

  IF w or h exceeds the device's maximum texture edge THEN RETURN   # caller degrades

  IF format is a depth format THEN
    device_format = the matching *uninterpreted* layout      # see Notes
    role = depth
  ELSE
    device_format = translate(format)
    role = colour

  IF NOT device.supports(device_format, as a 2D texture AND in that role) THEN RETURN

  IF flags has derived-from-the-presentation-chain AND role is colour THEN
    index = the number at the end of the name
    surface = presentation_chain.buffer(index)        # adopt, do not allocate

  IF no surface yet THEN
    evict resources to make room                       # see Notes
    describe: w, h, one mip level, `slices` array entries, device_format,
              `samples` samples, default usage,
              bindable as a shader resource AND in its role
    IF samples > 1 THEN
      IF the device tier is the oldest AND role is depth THEN
        drop the shader-resource binding   # that tier cannot sample a multisampled depth target
      IF the multisample-quality option is on THEN request the standard sample pattern
    IF shader-writable was requested AND the tier allows it AND
       there is no multisampling AND the format supports it THEN
      add the shader-writable binding
    surface = device.allocate(description)

  IF role is depth THEN
    view_description = { dimensionality from (slices > 1, samples > 1),
                         format = the concrete depth interpretation of device_format }
    depth_view_all = device.create_depth_view(surface, whole array)
    FOR slice IN 0 .. slices - 1
      depth_view_per_slice[slice] = device.create_depth_view(surface, that slice only)
    FOR EACH context: bound_depth_view = depth_view_all
  ELSE
    colour_view = device.create_colour_view(surface)

  IF shader-writable binding present THEN
    unordered_view = device.create_writable_view(surface)

  IF the surface is not bindable as a shader resource THEN RETURN   # e.g. an adopted back buffer
  texture = resources.create_texture(name) bound to this same surface
```

**Invariants** — **The allocation is uninterpreted and the views carry the interpretation.** A depth target is allocated with a layout that says only how many bits each component has; the depth view says "read these bits as depth-and-stencil" and the sampling view says "read them as a normalized value". Without this two-step, a depth buffer could not be sampled later in the frame, which the entire lighting pass depends on.

**Adopting a presentation-chain buffer is a create path, not a separate type.** A target flagged as derived from the chain takes the chain's buffer as its surface — parsing the buffer index out of the trailing digits of its own name — and then follows the identical view-creation path. It has no sampling texture, because chain buffers are not declared as shader-readable.

**Notes** — *Eviction before allocation.* The resource manager is asked to release unreferenced resources first. A render target is a large, contiguous allocation and the engine creates them in bursts at level load, when the texture set is also being uploaded.

*Failure is not fatal.* Returning without a surface leaves the target invalid, and the render-target set treats that as "this quality option is unavailable here" and falls back. That is what makes one renderer run across a wide hardware range, and a rebuild that throws on an unsupported format loses it.

*The multisampled-depth restriction.* On the oldest supported capability tier, a multisampled depth allocation cannot also be sampled, so the sampling binding is dropped and any technique that needed it must not be selected. The engine's quality-option logic checks the tier separately; this is the enforcement.

## `destroy`

**Contract** — releases the sampling texture's hold on the surface first, then the colour or depth views, then the surface, then the shader-writable view. Decrements the target-memory statistic.

## `reset_begin` / `reset_end`

**Contract** — the device-loss pair: destroy everything, then recreate from the stored name, dimensions, format, sample count, slice count and flags. This is why every one of those is a field: **the target is its own recreation recipe.** A rebuild must keep the same property — the seam requires a device-lost path that rebuilds every resource, and the only affordable way is for each resource to remember how it was made.

## `set_slice_read` / `set_slice_write`

**Contract** — choose which array slice the sampling side exposes, and which slice (or the whole array) a given submission context writes into. The write selection is per context, because two contexts may render different cascades of the same shadow map array in parallel.

## `resolve_into`

**Contract** — collapse a multisampled target into a single-sampled one of the same format. Requires the formats to match. Issued on the immediate context.

**Notes** — The source marks this as belonging in the command list rather than on the target, for the same reason as the stream buffers: it hard-codes the immediate context and so cannot be recorded by a worker.

## `create` (reference wrapper)

**Contract** — the reference-counted handle's creation path asks the resource manager for a target by name, so that two subsystems requesting the same named target share one.

**Notes** — A cube-map render target type exists in the source only as a commented-out body from the previous backend generation. This renderer never renders into a cube map; environment reflections are approximated instead. A rebuild adding real cube rendering is adding a feature, not restoring one.
