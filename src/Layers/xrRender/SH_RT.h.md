# src/Layers/xrRender/SH_RT.h

> A render target: a named, interned, device-owned surface that the frame graph draws into and later samples, together with the create/destroy/reset lifecycle every such surface shares.

**Needs** — [`SH_Texture.h`](SH_Texture.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`xrCore/xr_resource.h`](../../xrCore/xr_resource.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`ResourceManager_Reset.cpp`](ResourceManager_Reset.cpp.md) · [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md) · [`Shader.cpp`](Shader.cpp.md) · [`Shader.h`](Shader.h.md) · [`dx11SH_RT.cpp`](../xrRenderDX11/dx11SH_RT.cpp.md)
**Tier floor** — T1: it owns device surfaces and views of them, and it must release and rebuild them on demand when the device is lost.

## Purpose

The deferred renderer is a pipeline of passes that write into off-screen surfaces and read them back. A render target is one such surface, and this record is what makes it usable from both ends: it is a **draw destination** (a target view, or a depth view, or an unordered-access view) *and* a **texture** (bindable to a material stage by name).

It is declared in a header with no implementation file because each graphics backend implements the same record differently — the record's shape and contract are shared, its bodies are not. That makes this header the substance holder for the contract a backend must satisfy.

## State

```text
RECORD RenderTarget
  name        : text               # interned; the name a material binds it by
  texture     : Texture            # the sampling view of this surface, registered under the same name
  width, height : int
  format      : device format
  sample_count : int               # 1 = not multisampled
  slice_count  : int               # 1 = a plain surface; more = a texture array
  order        : int (64-bit)      # creation order, used to sequence teardown
  # device-side, per backend:
  surface            : device texture
  target_view        : device render-target view
  depth_views        : one per rendering context, plus an all-slices view and one view per slice
  access_view        : device unordered-access view, when requested
```

**Invariants**

- **The render target and its texture share a name.** Creating a target registers a texture of the same name in the resource registry, which is how a pass written in the shipped material scripts binds the output of an earlier pass by naming it. This is a frozen convention, and it is the single most important line on this page: the frame graph is wired together by *string*, not by handle.
- A target is valid exactly when its texture exists. There is no separate validity flag, and every path that releases the surface must also drop the texture.
- The creation order is recorded because targets are destroyed in a defined sequence; a target created later may hold a view onto one created earlier (a depth view shared across contexts, a slice of an array). A rebuild that tracks the dependency explicitly does not need the counter.
- Depth views are kept **per rendering context** rather than one per surface: the newest backend records commands on several contexts and a depth view is not shareable between them. On single-context backends the array has one entry.
- A multisampled target cannot be sampled directly; it must be resolved into a non-multisampled target of the **same format**. The resolve entry point asserts that and nothing checks anything else.

## `create`

**Contract** — allocate the surface and every view the flags ask for, and register the sampling texture under the same name. Takes name, dimensions, format, sample count, slice count and a flag set. Fatal if the device refuses. The flags are:

```text
create_access_view   # also make an unordered-access view; the compute paths need one
create_surface       # allocate a plain surface (depth-stencil or off-screen) instead of a texture;
                     # only meaningful on the oldest backend, where the two are different objects
create_base          # wrap the existing back buffer or its depth buffer rather than allocating;
                     # the format decides which of the two
```

**Notes** — `create_base` is what makes the final, presentable surface addressable by the same name-based machinery as every intermediate one. Without it the last pass of the frame would need a special case.

## `destroy`

**Contract** — release every view, the surface and the texture, and unregister. Idempotent.

## `reset_begin` · `reset_end`

**Contract** — the device-lost bracket. `reset_begin` releases everything the device owns while keeping the record, its name and its parameters; `reset_end` rebuilds from those same parameters. A target that survives a reset is the *same* record with the *same* name, so every material binding stays valid across the reset — which is the whole reason the bracket exists rather than a destroy-and-recreate.

**Invariants** — between the two calls the record is not valid and must not be bound. The renderer's reset path is responsible for bracketing every target, in creation order.

## `set_slice_read` · `set_slice_write`

**Contract** — select which slice of an array target is sampled, and which is drawn into by a given context. Selecting a slice swaps which view the texture and the target expose; selecting the "all" slice restores the whole-array views. The record remembers the current and previous selection so that a redundant selection costs nothing.

**Notes** — Array targets exist for the cascaded shadow maps and for the per-face rendering of point-light shadows: one surface, one name, many slices, each drawn in its own pass and sampled by index. A rebuild may model each slice as its own target instead; what it must preserve is that they are one allocation, because the lighting pass samples across slices.

## `resolve_into`

**Contract** — copy a multisampled target into a non-multisampled one, resolving the samples. Both must have the same format; nothing else is checked. Blocks only in the sense that the device schedules it.

## `valid`

**Contract** — true when the target has a surface. Used by the frame graph to skip passes whose targets the current settings did not allocate.

## `ref_rt` — the counted handle

**Contract** — the handle carries the `create` call rather than the record, so a target is brought into existence by creating *through the handle*. Releasing the last handle destroys the target. The two-argument and slice-count forms are the same call with a default.

**Notes** — Putting creation on the handle rather than on the record is how creation becomes an *interning* operation: the handle consults the registry first and only builds a new record on a miss. A rebuild expresses this as a factory on the registry and loses nothing.

## Could not recover

A cube-map render target record is present in full but disabled, left from the graphics generation before the current one. Nothing references it, and the deferred renderer's point-light shadows use an array target with six slices instead. There is no record of whether the cube form was faster on hardware of that era or merely more convenient.
