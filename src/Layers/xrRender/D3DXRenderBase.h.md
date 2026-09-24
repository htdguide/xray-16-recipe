# src/Layers/xrRender/D3DXRenderBase.h

> The half of the renderer interface every backend shares: device lifetime, the frame bracket, gamma, resource management, and the pool of parallel render contexts.

**Needs** — [`xrEngine/Render.h`](../../xrEngine/Render.h.md) · [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) · [`r__sector.h`](r__sector.h.md) · [`xr_effgamma.h`](xr_effgamma.h.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`D3DXRenderBase.cpp`](D3DXRenderBase.cpp.md)
**Tier floor** — T1: it owns device buffers and a fixed pool of contexts indexed by a bit mask.

## Purpose

Declares the surface implemented in [`D3DXRenderBase.cpp`](D3DXRenderBase.cpp.md), and — more importantly — declares the **context pool**, which is inline here because it is on the hot path of every parallel render task.

Each of the three shipped renderer generations subclasses this and fills in the parts that differ. What is here is everything that does *not* differ: if a rebuild wants one renderer rather than three, this class plus the generation it keeps is the whole renderer's top level.

## The context pool — the load-bearing declaration

```text
A RENDER CONTEXT is one independent scene-graph accumulation: a visible set
being gathered and a command list being recorded. The newest renderer
generation gathers several at once — the main view and each shadow cascade —
on separate workers.

RECORD ContextPool
  contexts : list<SceneGraphContext> of fixed length       # one immediate + N parallel
  in_use   : bit set over that length
```

```text
FUNCTION allocate_context(with_own_command_list) -> context id or none
  IF every context is in use   RETURN none
  id = the index of the first clear bit
  mark it used; reset the context; stamp its id
  its command list is either its own, or the immediate one       # see below

FUNCTION release_context(id)
  REQUIRE id is not the immediate context      # the immediate one is never released
  clear its bit

FUNCTION immediate_context() -> the fixed context at index zero
FUNCTION cleanup_contexts()  -> reset every context and clear the whole bit set
```

**Invariants**

- **Context zero is the immediate context** and is never released. It is the one whose command list goes straight to the device; the others record.
- A context may be allocated *without* its own command list, in which case it records into the immediate one. That is how a parallel *visibility* gather feeds a serial draw — the gather is the expensive half, and splitting it from recording is worth doing even when recording cannot be parallelized. A rebuild on an API without deferred command lists takes exactly this path for everything.
- Allocation fails by returning a sentinel rather than blocking or growing. The caller — the frame's top level — knows how many contexts it needs and the pool is sized for it; a failure means a bug, not contention.
- `cleanup_contexts` runs at the **end of every frame**, unconditionally, and resets everything. Contexts do not survive a frame boundary. This is what makes a leaked context a one-frame problem instead of a permanent one, and it is why allocation never needs to reclaim.
- The oldest renderer generation has exactly one context and the whole pool collapses to a single record. The two shapes are compiled alternatives in the original; a rebuild has one pool whose size happens to be one.

## Exported units

- **`set_gamma` / `set_brightness` / `set_contrast` / `update_gamma`** — the display transfer curve; see [`xr_effgamma.cpp`](xr_effgamma.cpp.md).
- **`create` / `destroy` / `on_device_create` / `on_device_destroy` / `reset`** — the device lifecycle.
- **`obtain_required_window_flags`** — what the renderer needs from the window before it exists.
- **`setup_states`** — push the initial render state after capabilities are known.
- **`begin` / `clear` / `end` / `clear_target`** — the frame bracket.
- **`set_cache_transform`** — push the view and projection to every context.
- **Resource management** — deferred load and upload, memory accounting, the texture store/destroy pair used around a level change.
- **Device state queries** — lost/ready, forced-reference-device, the frame's polygon count.
- **`overdraw_begin` / `overdraw_end`** — the overdraw visualization mode.
- **`get_imgui_texture`** — resolve a texture by name into what the debug overlay needs.
- **`dump_statistics`** — the developer overlay's render page.
- **`create_quad_index_buffer`** — the shared index buffer every quad batch in the engine draws through.

## Owned state

```text
resources        : the resource manager (materials, textures, shaders)
vertex_stream,
index_stream     : the shared dynamic geometry streams
quad_index_buffer: a static index buffer of 0,1,2 / 2,3,0 repeated
wire_shader,
selection_shader,
portal_fade_shader + its geometry : four fixed materials the renderer itself needs
gamma            : the display transfer curve
loaded           : whether a level's render data is up
```

**Invariants** — The quad index buffer is built once and shared by *everything* that draws quads: the user interface, particles, decals, post-processing. Every quad batch writes only vertices and draws through this buffer at an offset. It is the reason none of those paths has an index buffer of its own.
