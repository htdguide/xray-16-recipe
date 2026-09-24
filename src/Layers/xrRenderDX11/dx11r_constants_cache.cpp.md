# src/Layers/xrRenderDX11/dx11r_constants_cache.cpp

> Routes a constant write to the right block for the right stage, and uploads every bound block at draw time.

**Needs** — [`dx11r_constants_cache.h`](dx11r_constants_cache.h.md) · [`dx11ConstantBuffer.h`](dx11ConstantBuffer.h.md) · [`xrRender/r_constants_cache.h`](../xrRender/r_constants_cache.h.md)
**Used by** — [`dx11r_constants_cache.h`](dx11r_constants_cache.h.md)
**Tier floor** — T1: it decodes a packed bit field into an array index on every constant write.

## Purpose

The constant cache is one indirection: *name-resolved constant* → *the block object currently bound to that stage at that slot*. This file is the resolution step and the flush step.

## `block for a stage`

**Contract** — given a constant record and a stage, extracts that stage's block index from the constant's packed destination field and returns the block the command list currently has bound at that index. Asserts the index is in range and that a block is actually bound — writing a constant whose block is not bound is a programming error, not a recoverable condition, because it means the material's constant table and the bound program disagree.

There is one of these per stage. They differ only in which bit field they decode, which is why the source expresses them as six specializations of one function and forbids a generic fallback: a stage that is added without its decoder is a compile error rather than a silent mis-bind.

## `flush_cache`

**Contract** — uploads every block bound at every slot of every stage that is dirty, on the command list's own submission context. Called once per draw and once per dispatch, after state has been applied.

```text
FUNCTION flush()
  FOR EACH slot IN 0 .. max_blocks_per_stage - 1
    FOR EACH stage IN {vertex, pixel, geometry, hull, domain, compute}
      IF a block is bound at (stage, slot) THEN block.flush(context)
```

**Notes** — The iteration is unconditional: there is no per-frame dirty list, and a block that was not written simply finds itself clean and returns. The cost is one flag test per bound block per draw, which is why the loop is written flat rather than over a collection.

The previous generation of this code, preserved in the source as a comment, took a different shape worth knowing about because a rebuild on an older-style API may need it: there were exactly **two** arrays — one for vertex, one for pixel — of 256 four-component registers, with a dirty *range*, and a flush uploaded the whole 256-register array. The move to named blocks replaced a register file with a set of independently uploadable buffers, which is why the range tracking disappeared and why the dedup in [`dx11ConstantBuffer.cpp`](dx11ConstantBuffer.cpp.md) became worth doing.
