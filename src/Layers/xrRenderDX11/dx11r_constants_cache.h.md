# src/Layers/xrRenderDX11/dx11r_constants_cache.h

> The write side of the constant system: one call fans a named value out to every stage that declared it.

**Needs** — [`dx11r_constants_cache.cpp`](dx11r_constants_cache.cpp.md) · [`dx11ConstantBuffer.h`](dx11ConstantBuffer.h.md) · [`xrRender/r_constants.h`](../xrRender/r_constants.h.md)
**Used by** — [`dx11DetailManager_VS.cpp`](dx11DetailManager_VS.cpp.md) · [`dx11R_Backend_Runtime.h`](dx11R_Backend_Runtime.h.md) · [`dx11r_constants_cache.cpp`](dx11r_constants_cache.cpp.md)
**Tier floor** — T1: every setter must inline; the fan-out is a run of bit tests on the hot path.

## Purpose

This is the object the renderer actually writes constants through, and the reason a caller never mentions a shader stage. A constant record carries a set of stage flags; a single `set` call tests each flag and writes the value into that stage's block. The material system's `set("fog_color", …)` therefore reaches the vertex program's copy and the pixel program's copy in one call, at their respective offsets, with no knowledge of either.

It is declared as a substantive header (rather than an implementation file) because the fan-out must inline: the alternative is a function call per stage per constant per draw.

## State

```text
RECORD ConstantCache
  cmd_list : CommandList   # the list whose bound blocks this cache writes into
```

Stateless otherwise — it owns nothing. The blocks it writes into belong to the currently bound constant table, which is why swapping tables invalidates every cached write pointer (see [`dx11R_Backend_Runtime.h`](dx11R_Backend_Runtime.h.md)).

## `set` / `seta`

**Contract** — write a value (matrix, vector, scalar float, scalar integer) to a named constant, or to one element of an array constant. Fans out over the constant's stage flags: for each stage the constant is declared in, resolve that stage's block and write at that stage's offset. Writing a constant that no stage declares is a no-op, which is deliberate — a material may set a quantity a particular pass's shader does not use.

```text
FUNCTION set(constant, value)
  FOR EACH stage IN {pixel, vertex, geometry, hull, domain, compute}
    IF constant.destination has stage THEN
      block = block_bound_at(stage, constant.block_index_for(stage))
      block.write(constant, constant.load_for(stage), value)
```

A convenience form takes four loose floats and packs them into a vector first; the engine's older call sites are written that way.

## `access_direct`

**Contract** — hands back raw write pointers into the vertex, geometry and pixel copies of a named constant, or nothing per stage where it is absent, for a caller that will fill a span itself. Deliberately covers only those three stages: the direct-write callers are post-process and volumetric passes, which never use the tessellation stages.

## `flush`

**Contract** — uploads every dirty bound block; see [`dx11r_constants_cache.cpp`](dx11r_constants_cache.cpp.md).

**Notes** — The stage enumeration here is an ordering, not an identity: the stage flags and the per-stage block-index fields live in the constant's packed destination word, and this type's stage list must agree with that packing. In a rebuild they should be one declaration.
