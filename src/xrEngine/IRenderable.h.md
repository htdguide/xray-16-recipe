# src/xrEngine/IRenderable.h

> What an object must answer before the renderer will put it in a frame.

**Needs** — [`IRenderable.cpp`](IRenderable.cpp.md) · [`Render.h`](Render.h.md) · [`xrCDB/ISpatial.h`](../xrCDB/ISpatial.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r__dsgraph_build.cpp`](../Layers/xrRender/r__dsgraph_build.cpp.md) · [`r__dsgraph_render.cpp`](../Layers/xrRender/r__dsgraph_render.cpp.md) · [`IRenderable.cpp`](IRenderable.cpp.md) · [`PS_instance.h`](PS_instance.h.md) · [`Render.h`](Render.h.md) · [`xr_object.h`](xr_object.h.md) · [`base_client_classes_wrappers.h`](../xrGame/base_client_classes_wrappers.h.md)
**Tier floor** — T1: the record is read by the renderer every frame for every visible object, so its layout and its cost of access are load-bearing

## Purpose

This is one of the three facets every drawable game object wears — *spatial* (where it is,
for culling), *scheduled* (when it is updated), *renderable* (how it is drawn). The facet
is separated from the object because the renderer must be able to walk a list of
renderables without knowing what game types they are, and because an object can be
spatial without being renderable (a sound source) or renderable without being scheduled
(a static decal holder).

The interface is substantive: what it demands *is* the contract a rebuild must satisfy.
The shared implementation of the easy half lives in [`IRenderable.cpp`](IRenderable.cpp.md).

## State

The record every renderable carries. The renderer reads all six fields per visible object
per frame, so they are one contiguous block rather than a set of virtual calls.

```text
RECORD RenderData
  xform         : matrix4          # world transform used for this frame's draw
  visual        : optional<RenderVisual>   # the mesh/particle/skeleton asset; none = nothing to draw
  ros           : optional<ObjectSpecific> # per-object lighting cache; created lazily
  ros_allowed   : bool             # false forbids ever creating one (particles, HUD-only items)
  invisible     : bool             # excluded from the scene graph this frame
  hud           : bool             # currently being drawn in the first-person overlay pass

# invariant: ros is none whenever ros_allowed is false
# invariant: visual is none or owned by the renderer, never by the object
```

## `IRenderable`

**Contract** — the demands on an implementor. Every call is per-frame hot; none may block
or allocate except `renderable_ros`, which allocates at most once per object.

```text
INTERFACE Renderable
  get_render_data() -> RenderData          # by reference; the renderer writes xform through it
  render(context_id : int, root : Renderable)
  renderable_ros() -> optional<ObjectSpecific>
  shadow_generate() -> bool                # does this object cast?
  shadow_receive()  -> bool                # does it take a projected shadow?
  invisible()       -> bool
  set_invisible(bool)
  hud()             -> bool
  set_hud(bool)
```

`render` is the object's chance to submit *additional* visuals beyond its own — an
attached weapon, a carried item, a muzzle flash. `root` is the top-level renderable the
traversal started from, which differs from `self` exactly when this object is being drawn
as somebody's attachment; children use it to inherit the root's lighting cache rather than
computing their own. `context_id` names which of the renderer's parallel recording
contexts to submit into, because visibility for several views (main camera, a second
viewport, shadow cascades) is gathered concurrently.

**Notes** — the shadow predicates are queries rather than flags in the record because they
are usually derived: a corpse stops casting when it falls below a size threshold, an
invisible object still casts if it is a light-blocker. Making them virtual costs one call
per object per shadow pass and buys the game module the ability to decide per frame.

## `RenderableBase`

**Contract** — the shared filling almost every implementor uses: it owns a `RenderData`,
answers the accessors directly from it, and defaults both shadow predicates to false so a
new object type is invisible to the shadow passes until it opts in. Its construction and
destruction decisions are in [`IRenderable.cpp`](IRenderable.cpp.md).

**Notes** — the original marks this class as *not* devirtualizable specifically because
its constructor performs a downward type test on itself (see the `.cpp`), which is
incidental: a rebuild that passes the spatial facet in explicitly does not need the test
and does not pay for the vtable.
