# src/Include/xrRender/UIShader.h

> A material handle for two-dimensional drawing: the pairing of a named material pass description with a named texture.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`xrEngine/Render.h`](../../xrEngine/Render.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`DebugShader.h`](DebugShader.h.md) · [`RenderFactory.h`](RenderFactory.h.md) · [`UIRender.h`](UIRender.h.md) · [`UISequenceVideoItem.h`](UISequenceVideoItem.h.md) · [`WallMarkArray.h`](WallMarkArray.h.md) · [`dxUIShader.cpp`](../../Layers/xrRender/dxUIShader.cpp.md) · [`dxUIShader.h`](../../Layers/xrRender/dxUIShader.h.md) · [`HitMarker.cpp`](../../xrGame/HitMarker.cpp.md) · [`Tracer.cpp`](../../xrGame/Tracer.cpp.md) · [`map_spot.cpp`](../../xrGame/map_spot.cpp.md) · [`UIPdaKillMessage.cpp`](../../xrGame/ui/UIPdaKillMessage.cpp.md) · [`UIStatsIcon.cpp`](../../xrGame/ui/UIStatsIcon.cpp.md) · [`UIProgressShape.cpp`](../../xrUICore/ProgressBar/UIProgressShape.cpp.md) · [`UIFrameLineWnd.cpp`](../../xrUICore/Windows/UIFrameLineWnd.cpp.md) · _and 2 more_
**Tier floor** — T2: an opaque handle with a name-based create; the only thing tying it down is that it hands out a device texture identifier for the debug overlay to draw with.

## Purpose

Everything the UI draws needs a material — how to blend, how to test depth, which texture to sample. In this engine a *shader* in that sense is a **material pass description loaded from data**, not a program: a named entry in the game's material files that says which passes to run with which state and which sampler settings. This interface is the handle to one of those, resolved and bound to a particular texture.

It is created through the [render factory](RenderFactory.h.md) and held by value through [`FactoryPtr.h`](FactoryPtr.h.md), so every UI element that draws owns one as a field.

## State

```text
RECORD UIShader
  resolved : optional<MaterialPassChain>   # none until create() succeeds
```

**Invariants** — a freshly created handle is empty and drawing with it is an error; `inited` is how a caller checks. The handle may be re-created, which releases the previous binding.

## `IUIShader` — what an implementor must provide

### `create(material_name, texture_name)`

**Contract** — resolves a named material against a named texture and binds the result. The texture name may be omitted, in which case the material's own texture bindings stand. Both names are logical paths into the virtual filesystem, without extensions. Loading is deferred by the renderer's resource system: creating a handle does not necessarily touch the disk or the device, and the texture may still be uploading when the first frame draws it.

### `destroy`

**Contract** — releases the binding, returning the handle to the uninitialized state. Idempotent.

### `inited`

**Contract** — whether the handle currently names a material. Callers check it before every draw, because UI elements are constructed before their textures are named.

### `copy(other)`

**Contract** — makes this handle name the same material and texture as another. This is the hook that [`FactoryPtr.h`](FactoryPtr.h.md)'s copy policy calls; it must be a *value* copy in effect, though implementations share the underlying resolved material by reference count.

### equality

**Contract** — two handles are equal when they name the same resolved material. Used by the widget toolkit to avoid redundant material changes when drawing a run of elements.

### `base_texture_size`

**Contract** — the pixel dimensions of the material's first texture, or nothing if it has none or it has not loaded yet. The toolkit uses it to size an element to its art without the layout author repeating the numbers, and to convert authored pixel rectangles into normalized texture coordinates.

**Notes** — This is the one place the UI reaches through the material to the texture underneath, and it is the reason a material handle cannot be fully opaque. It also forces the resolution to be knowable without the texture being resident, which the renderer satisfies by keeping dimensions in its texture descriptor rather than in the uploaded resource.

### `imgui_texture_id`

**Contract** — the material's first texture as an identifier the debug overlay toolkit can put in its own draw lists, together with the texture's size. The overlay draws through the engine's device but builds its own geometry; this is the handshake.

**Notes** — This method is why the debug overlay seam is *given* rather than pluggable: the toolkit defines the identifier's type, so the engine's material handle has to speak it. A rebuild that drops the debug overlay drops this method with it; a rebuild that keeps it must accept the toolkit's identifier type reaching into the material interface, or add an adapter layer here.
