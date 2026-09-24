# src/Include/xrRender/UIRender.h

> The immediate-mode vertex sink every two-dimensional thing in the game draws through: menus, the heads-up display, the map, crosshairs, damage indicators.

**Needs** — [`UIShader.h`](UIShader.h.md) · [`xrAPI/xrAPI.h`](../xrAPI/xrAPI.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`xrAPI.h`](../xrAPI/xrAPI.h.md) · [`dxUIRender.cpp`](../../Layers/xrRender/dxUIRender.cpp.md) · [`dxUIRender.h`](../../Layers/xrRender/dxUIRender.h.md) · [`EliteDetector.cpp`](../../xrGame/EliteDetector.cpp.md) · [`Tracer.cpp`](../../xrGame/Tracer.cpp.md) · [`UIGameTutorialVideoItem.cpp`](../../xrGame/ui/UIGameTutorialVideoItem.cpp.md) · [`UIProgressShape.cpp`](../../xrUICore/ProgressBar/UIProgressShape.cpp.md) · [`UIStaticItem.cpp`](../../xrUICore/Static/UIStaticItem.cpp.md) · [`UIFrameLineWnd.cpp`](../../xrUICore/Windows/UIFrameLineWnd.cpp.md) · [`UIFrameWindow.cpp`](../../xrUICore/Windows/UIFrameWindow.cpp.md) · [`ui_defs.h`](../../xrUICore/ui_defs.h.md)
**Tier floor** — T1: the caller pushes vertices one at a time directly into a mapped device buffer whose layout is chosen by an enumerated vertex type; the buffer's byte stride and the vertex's field order are the contract.

## Purpose

The widget toolkit and the game's screens know how to lay themselves out and what texture each piece wants; they know nothing about buffers, state objects or draw calls. This interface is the whole of what they are given: choose a material, declare how many vertices are coming and in what topology, push them, flush. One instance exists per process, published through the global environment when a renderer is selected.

It is an *immediate* interface on purpose. The UI is rebuilt from scratch every frame — there is no retained geometry, no dirty tracking, no vertex caching — because the amount of UI on screen is small and the layout logic is script- and XML-driven and changes constantly.

## State

The interface is a facade over renderer state; an implementor holds a little.

```text
RECORD UIRenderState
  geometry_tl   : GeometryBinding    # for pre-transformed vertices
  geometry_lit  : GeometryBinding    # for vertices that still need a transform
  current_type  : ENUM PointType     # none between primitives
  current_topo  : ENUM Topology      # none between primitives
  write_cursor  : position in a mapped device buffer
  capacity      : int                # the max_vertices the caller promised
```

**Invariants** — a primitive is open between `begin_primitive` and `flush_primitive` and only then; pushing outside that window is an error. The caller must not push more vertices than it promised. Both bindings share the renderer's general-purpose dynamic vertex buffer.

## The two vertex kinds

```text
ENUM PointType
  pre_transformed     # position is already in screen pixels with a depth; no transform applied
  lit                 # position is in the current world transform's space
```

This is the interface's one real decision. Nearly all UI is drawn pre-transformed — the widget already knows its pixel rectangle, and pushing it through a projection would only undo itself. But some UI is drawn in the world: a map marker on a rotating minimap, a damage arrow that points at a world direction, the crosshair's dynamic spread. Those need a transform, so they use the second kind and set a world matrix first.

A rebuild targeting a device without a pre-transformed vertex path implements it as an orthographic projection matched to the framebuffer, taking care that the pixel-centre convention matches, or the whole UI shifts half a pixel and every sharp edge blurs.

```text
ENUM Topology
  triangle_list
  triangle_strip
  line_strip
  line_list
```

Four topologies, no indexed path. UI geometry is small enough that repeating shared vertices is cheaper than maintaining an index buffer.

## `IUIRender` — what an implementor must provide

### `create_geometry` / `destroy_geometry`

**Contract** — bind and release the two vertex layouts against the renderer's dynamic vertex buffer. Called from the device creation and destruction paths, and again around a device reset. Nothing may be drawn between a destroy and the next create.

### `set_shader`

**Contract** — makes a material the active one for subsequent primitives. The material comes from a [`UIShader`](UIShader.h.md) the caller owns. Must be called before the first push of a primitive; changing it mid-primitive is undefined.

### `set_alpha_reference`

**Contract** — sets the alpha-test cutoff for subsequent primitives. Used by the toolkit to make partially transparent UI textures cut cleanly rather than blend.

### `set_scissor`

**Contract** — restricts subsequent drawing to a rectangle in screen pixels, or clears the restriction when given nothing. This is how a scrolling list clips its contents. It must survive the material changes that happen inside a clipped region, so it is renderer state and not part of a primitive.

**Notes** — On the newer backends this also has to force scissoring *on* in the state object, because the material's own state block may have disabled it. That is a device detail, but the decision behind it is portable: **the scissor set here overrides whatever the material asked for**, because the widget that set it is the outer scope.

### `begin_primitive` / `push_point` / `flush_primitive`

**Contract** — the drawing bracket.

```text
FUNCTION begin_primitive(max_vertices : int, topology, point_type)
  # Reserves max_vertices from the shared dynamic vertex buffer, maps it,
  # and remembers the topology so flush can issue the right draw.

FUNCTION push_point(x, y, z : real, colour : int (32-bit), u, v : real)
  # Appends one vertex. The field set is fixed: position, packed colour, one texture coordinate pair.

FUNCTION flush_primitive()
  # Unmaps, binds the geometry for the chosen point type, and issues one draw
  # of however many vertices were actually pushed. Pushing none draws nothing.
```

**Invariants** — the reservation is an upper bound, not an exact count; callers routinely reserve for the worst case and push fewer. The vertex buffer is shared with the rest of the renderer, so a primitive must be flushed before anything else draws.

**Notes** — Reserving before pushing is what keeps this cheap: one buffer map per primitive rather than per vertex, with the discard hint so the device never waits on the previous frame's reads. A rebuild that builds the vertices into its own array and uploads at flush time is equivalent and simpler; what must survive is that **the caller declares its vertex count up front**, because the whole UI's worth of draws must fit one dynamic buffer without stalling.

A vertex's colour is one packed 32-bit value, not four floats. That packing is part of both vertex layouts and is therefore visible to the shaders that ship with the game.

### `update_shader_name`

**Contract** — given a texture name and a material name, returns the material name that should actually be used. It answers one question: *does a video file exist under this texture's name?* If so, the caller gets the movie-playback material instead of the one it asked for, and the texture will be supplied by a video decoder rather than an image.

**Notes** — This is how a UI element becomes a video screen without the toolkit knowing what video is: a screen names a "texture", and if content shipped a video by that name, it plays. A rebuild should keep the *substitution* and move the decision to wherever it resolves texture names, because as written it also silently falls back to the requested material on devices too old for the movie shader — a capability check that no longer discriminates anything.

### `set_world_transform` / `set_cull_mode`

**Contract** — the two pieces of renderer state the untransformed vertex kind needs. Culling has three settings — off, clockwise, counter-clockwise — and UI code sets it off far more often than not, because two-dimensional geometry is routinely wound either way.

## Notes

The header carries a large block of commented-out methods: per-topology start/flush pairs, integer and two-dimensional point pushes. They were collapsed into the single parameterized bracket above, and the collapse is the better design — a rebuild should not resurrect them. Their presence is useful only as evidence that the topology set is closed and was arrived at by elimination.

There is no `destroy` on the interface and no reference counting: the single instance is owned by the renderer module and published into the global environment for the process's lifetime.
