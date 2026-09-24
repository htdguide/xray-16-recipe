# src/Layers/xrRender/HWCaps.h

> The device capability record: what the renderer is allowed to assume about the hardware it found, filled in once per backend at startup.

**Needs** — [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`D3DXRenderBase.cpp`](D3DXRenderBase.cpp.md) · [`DetailManager_VS.cpp`](DetailManager_VS.cpp.md) · [`DetailModel.cpp`](DetailModel.cpp.md) · [`Blender_LaEmB.cpp`](blenders/Blender_LaEmB.cpp.md) · [`Blender_Lm(EbB).cpp`](blenders/Blender_Lm%28EbB%29.cpp.md) · [`Blender_Screen_SET.cpp`](blenders/Blender_Screen_SET.cpp.md) · [`r__sync_point.cpp`](r__sync_point.cpp.md) · [`r__sync_point.h`](r__sync_point.h.md) · [`stats_manager.cpp`](stats_manager.cpp.md) · [`dx11HW.cpp`](../xrRenderDX11/dx11HW.cpp.md) · [`dx11HW.h`](../xrRenderDX11/dx11HW.h.md) · [`dx11HWCaps.cpp`](../xrRenderDX11/dx11HWCaps.cpp.md) · [`glHW.h`](../xrRenderGL/glHW.h.md) · [`glHWCaps.cpp`](../xrRenderGL/glHWCaps.cpp.md)
**Tier floor** — T1: it is the shape of a struct the backends fill and the render path branches on per frame; several fields are packed bit-fields sized to the values they hold.

## Purpose

Declares the one record that answers "can the machine do this?" for the whole renderer, so that no render path queries the device directly. Each backend fills it at startup; everything above reads it. This is the only file where capability *vocabulary* lives, and it is shared because the two backends must answer the same questions even though they ask their drivers very differently.

The record is a 2007 artefact in its shape and largely a constant in its content. Reading the two backends' fillings side by side is instructive: the OpenGL one hard-codes almost every field, and the Direct3D one queries only vendor identity and the multi-GPU count. The capability *variation* this record was designed to express — shader model, register counts, texture stage counts — no longer varies on any machine the engine runs on. A rebuild should keep the handful of fields that are still live and delete the rest; they are named below.

## State

```text
RECORD DeviceCaps
  # --- shader capability, effectively constant on any supported machine ---
  geometry_major, geometry_minor : int      # vertex shader generation
  geometry_profile               : text     # the compile target name passed to the shader compiler
  raster_major,  raster_minor    : int      # pixel shader generation
  raster_profile                 : text

  geometry : RECORD
    registers, instructions : int (16-bit each)
    software                : bool     # vertex processing runs on the CPU
    point_sprites           : bool
    vertex_texture_fetch    : bool     # LIVE: gates the tree-wave and terrain paths
    n_patches               : bool
    clip_planes             : int (4-bit)
    vertex_cache            : int (8-bit)   # LIVE: the stripifier optimizes for this size

  raster : RECORD
    registers, instructions : int (16-bit each)
    stages                  : int (4-bit)   # fixed-function texture stages
    mrt_count               : int (4-bit)   # LIVE: how many render targets bind at once
    mrt_mixed_depth         : bool          # LIVE: may the bound targets differ in bit depth
    non_power_of_two        : bool
    cubemap                 : bool

  # --- identity and topology ---
  id_vendor, id_device : int              # LIVE: vendor-specific workarounds hang off these
  gpu_count            : int              # LIVE: drives the frame-latency depth; see below

  # --- framebuffer ---
  target_format, depth_format : texture format
  refresh_rate                : int

  # --- fixed-function survivals ---
  has_stencil, has_scissor, has_table_fog : bool
  stencil_increment_op, stencil_decrement_op : stencil operation
  max_stencil_value : int
  max_fixed_function_lights : int

  # --- structural, and genuinely load-bearing ---
  has_fixed_pipeline   : bool   # LIVE
  combined_samplers    : bool   # LIVE: see below
  force_reference_device, force_software, force_non_pure : bool   # debug overrides
```

**Invariants**

- If the pixel shader generation is zero, the vertex shader generation is forced to zero too. A machine that cannot run a pixel shader must not be handed a vertex shader either, because every material in the shipped data pairs them and a half-programmable pipeline has no valid path through the material system.
- The saturating stencil increment and decrement operations are chosen, not assumed, and the maximum stencil value is recorded alongside them. Stencil-volume shadowing counts in and out of a volume and *must* saturate rather than wrap: a wrap at the ceiling turns a shadowed pixel lit. The value is eight bits on both shipped backends.
- `combined_samplers` records whether the device treats a texture and its sampler settings as one object or two. This is the one field in the record that changes the *shape* of the material system's work rather than merely enabling a feature: with combined samplers, a texture bound at two different filter settings is two objects, and the pass compiler must produce two bindings. Both shipped backends answer true, but on the newer graphics APIs a rebuild targets, the answer is false, and this is the field that tells the material system so.
- `gpu_count` is not informational. The renderer uses it to size the depth of its frame pipelining — how many frames' worth of dynamic buffers and query results are in flight before it blocks. Getting it wrong costs either latency or a stall. The two backends discover it very differently: one asks the vendor's own multi-GPU library and the other simply answers two.

## `update()`

**Contract** — implemented once per backend, not here. Fills every field from the live device, before any resource is created and before any material compiles. Logs what it found. Must not fail: a device that got far enough to be queried is a device the renderer will use, and an unsupported one is rejected earlier, at device creation.

**Notes** — Answering a hard-coded "two" for the GPU count on one backend, with no discovery at all, has no recoverable justification. It is not a capability reading; it is a pipelining depth chosen by hand and parked in a field named after something else. A rebuild should make the frame-pipelining depth its own setting and let the capability record report only what it actually measured.

The vendor-specific multi-GPU discovery on the other backend links two proprietary vendor libraries to answer one integer. That is an optional dependency for a value with a sane default, and a rebuild can drop both.

## The eight-GPU ceiling

**Contract** — a compile-time maximum on how many physical devices the capability query will consider.

**Notes** — It bounds a stack array in the vendor query and reaches nothing else. It is pure arithmetic about an era in which multi-GPU rendering was a shipping feature; no current configuration exceeds one.
