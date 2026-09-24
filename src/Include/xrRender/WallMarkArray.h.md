# src/Include/xrRender/WallMarkArray.h

> A surface material's set of decal variants, and the draw from it: one bullet hole chosen at random out of the several a material declares.

**Needs** — [`FactoryPtr.h`](FactoryPtr.h.md) · [`UIShader.h`](UIShader.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`dxWallMarkArray.cpp`](../../Layers/xrRender/dxWallMarkArray.cpp.md) · [`dxWallMarkArray.h`](../../Layers/xrRender/dxWallMarkArray.h.md) · [`physics_game.cpp`](../../xrGame/physics_game.cpp.md) · [`GameMtlLib.h`](../../xrMaterialSystem/GameMtlLib.h.md) · [`GameMtlLib_Engine.cpp`](../../xrMaterialSystem/GameMtlLib_Engine.cpp.md)
**Tier floor** — T2: a small owned collection with a random pick; nothing device-facing crosses the interface.

## Purpose

A *wallmark* is a decal — a bullet hole, a blood splatter, a scorch. Which one appears depends on what was hit: the surface material library names, per material and per event kind, a comma-separated list of decal textures. Concrete gets chips, metal gets sparks and dents, flesh gets blood.

This interface is that list, owned by the material that declared it, living on the renderer's side because each entry is a resolved material. It is created through the [render factory](RenderFactory.h.md) and held by the material library and by any game object with its own decal set — a creature carries two, for blood marks and blood drops.

## State

```text
RECORD WallMarkArray
  variants : list<Material>     # one per texture named in the material's decal list
```

**Invariants** — populated once, when the material library or the owning object loads, and not modified afterwards. An empty array is normal and means "this surface takes no marks".

## `IWallMarkArray` — what an implementor must provide

### `append(texture_names)`

**Contract** — resolves one decal texture against the engine's fixed decal material and appends it. Called once per name while parsing the material library's decal list. Allocates.

**Notes** — The decal material is a single hard-coded name, the same for every wallmark in the game; only the texture varies. That is worth stating because it means **a rebuild needs exactly one decal pass**, and all the variety in shipped content is texture variety.

### `clear`, `empty`

**Contract** — discard the variants; ask whether there are any. The emptiness check is on the hot path: the collision code asks it before doing any of the work of placing a mark.

### `copy(other)`

**Contract** — the duplication hook [`FactoryPtr.h`](FactoryPtr.h.md)'s copy policy calls.

### `pick`

**Contract** — returns a freshly created material handle naming one variant chosen uniformly at random, or an empty handle if there are none.

**Notes** — This call carries a warning in the source, and the warning is the interesting part: **it allocates a new handle on every call, and every bullet impact calls it.** The renderer therefore offers a second path that takes the whole array and does the pick internally against its own already-resolved materials, and the engine's decal submission prefers that path. The single-pick form survives for callers that need to *hold* the chosen variant rather than draw it immediately.

The decision a rebuild should take from this: the random choice belongs on the drawing side, not the calling side, so that choosing a variant costs an index rather than an object. As written, the interface offers both and the comment is the only thing steering callers to the right one.

Uniform random across variants is the whole selection policy — there is no weighting, no avoidance of the previously chosen mark, no dependence on the projectile. Two identical bullet holes side by side are possible and are visible in the original.
