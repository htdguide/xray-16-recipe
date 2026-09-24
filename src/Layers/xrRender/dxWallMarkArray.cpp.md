# src/Layers/xrRender/dxWallMarkArray.cpp

> A surface material's set of decal variants, and the pick from it: one bullet hole chosen uniformly at random out of the several a material declares.

**Needs** — [`Include/xrRender/WallMarkArray.h`](../../Include/xrRender/WallMarkArray.h.md) · [`dxWallMarkArray.h`](dxWallMarkArray.h.md) · [`dxUIShader.h`](dxUIShader.h.md) · [`Shader.h`](Shader.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dxWallMarkArray.h`](dxWallMarkArray.h.md)
**Tier floor** — T2: an owned collection of material references with a uniform random pick; nothing device-facing crosses the interface.

## Purpose

The surface-material table declares, per material, a list of decal textures — several variants of a bullet hole in concrete, several in wood. The game asks for "a wallmark for this material" at the moment a bullet lands and must get a different one each time. This file owns that list and that pick.

The decision worth recording is that the *material* is fixed and only the *texture* varies: every variant is the same decal material over a different texture, so the whole array costs one material's worth of state and the pick never changes a pass.

## State

```text
RECORD WallMarkArray
  variants : list<Material>   # each is the decal material over one variant texture
```

Invariants:

- Every entry is a reference taken at append time and released when the array is destroyed. The array owns its references; the compiled materials are shared with anything else naming them.
- The array may be empty, and an empty array is normal — most surface materials declare no decals at all. Every reader must handle it.

## `append_mark`

**Contract** — Given a texture name from the surface-material table, resolve the fixed decal material over that texture and append it. Blocks on resolution. Called once per declared variant while the material table loads.

```text
FUNCTION append_mark(texture_name : text)
  variants.append(resolve_material("effects/wallmark", texture_name))
```

**Notes** — The material name is a frozen constant of the engine: every wallmark in all three games is drawn with the one `effects/wallmark` material, and the shipped material file of that name defines the decal's blending, depth bias and sort order. A rebuild may not choose its own name here — the material file ships with the game data.

## `clear` / `empty`

**Contract** — Drop every variant; report whether any remain. Used when the surface-material table is reloaded.

## `generate_wallmark`

**Contract** — Return a fresh opaque material handle holding one variant picked uniformly at random, or an empty handle when the array is empty. Does not block. This is what the game calls; it never sees the list.

```text
FUNCTION generate_wallmark() -> MaterialHandle
  result = empty handle
  IF variants is not empty
    result.material = variants[random_index(0, variants.count)]
  RETURN result
```

**Invariants** — The pick is uniform and unconditioned: there is no memory of the last pick, so the same variant can come up twice in a row. That is deliberate — a cheap uniform draw over four or five variants is visually sufficient and costs nothing per impact.

## `generate_wallmark_direct`

**Contract** — The same uniform pick, returning the concrete material reference rather than an opaque handle, or nothing when the array is empty. Exists for the renderer's own decal batcher, which is inside this module and would only unwrap the handle again. Callers outside the module must use the opaque form.

## Lifetime

**Contract** — Teardown releases every variant reference explicitly. In the original this is a destructor; what survives is the requirement that the array's references are dropped at a defined time — the material table's unload — and not left to a collector, because the textures behind them are a meaningful share of a level's texture budget.
