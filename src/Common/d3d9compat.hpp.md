# src/Common/d3d9compat.hpp

> The enumerations and constants of a retired graphics interface, restated portably, because the game's shipped material descriptions are written in its vocabulary.

**Needs** — _(none)_
**Used by** — [`NVMeshMender.cpp`](NvMender2003/NVMeshMender.cpp.md) · [`NVMeshMender.h`](NvMender2003/NVMeshMender.h.md) · [`OGF_GContainer_Vertices.hpp`](OGF_GContainer_Vertices.hpp.md)
**Tier floor** — T1: it fixes the numeric values of enumerations that appear in shipped data files and in vertex-layout descriptions.

## Purpose

The game's material files — the text descriptions that name each pass's blend mode, depth
test, cull direction, sampler filtering and texture stage operations — were authored
against a specific 2002-era graphics interface and name its enumerators. Those files ship
with the game and cannot be rewritten. So the vocabulary has to survive even on platforms
where the interface never existed, and this file is that survival: the enumerations,
the constants, and the vertex-declaration record, with the same numeric values and no
functions at all.

This is the single largest compatibility hazard named in
[§5 Data and persistence](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence), seen from
below. A rebuild can either keep this vocabulary as its internal one and translate at the
device boundary — what this engine does — or translate the shipped material files at load
time. It cannot ignore them.

## State

The file is entirely constants. They fall into seven groups.

```text
# 1. Pixel formats
ENUM PixelFormat : int (32-bit)
  # uncompressed colour, depth and index formats numbered 20..117, with deliberate gaps
  # where formats were withdrawn; plus block-compressed and packed-video formats whose
  # values are FOUR-CHARACTER CODES rather than small integers
  #   e.g. the DXT1..DXT5 block formats, and the two packed luminance-chroma formats
  unknown = 0
  ...

# 2. Fixed-function pipeline state
ENUM RenderState        # ~90 members, numbered 7..184 with large gaps
ENUM TextureStageState  # ~20 members
ENUM TextureOperation   # ~26 members: how a texture stage combines its arguments
ENUM SamplerState       # ~14 members: filtering, addressing, mip bias, anisotropy

# 3. Comparison and blending vocabulary
ENUM CompareFunction    # never, less, equal, less-or-equal, greater, not-equal,
                        #   greater-or-equal, always — numbered 1..8
ENUM StencilOperation   # keep, zero, replace, saturating increment/decrement,
                        #   invert, wrapping increment/decrement — 1..8
ENUM BlendFactor        # 1..15
ENUM BlendOperation     # add, subtract, reverse subtract, min, max — 1..5
ENUM CullDirection      # none = 1, clockwise = 2, counter-clockwise = 3
ENUM FillMode, ShadeMode, SwapEffect, PrimitiveType

# 4. The vertex declaration
RECORD VertexElement                # 8 bytes, tightly packed
  stream       : int (16-bit)       # which vertex buffer this element is read from
  offset       : int (16-bit)       # byte offset within that buffer's stride
  element_type : int (8-bit)        # a DeclarationType: float2, float3, packed colour,
                                    #   two or four 16-bit integers, and so on
  method       : int (8-bit)        # a DeclarationMethod; always "default" in this engine
  usage        : int (8-bit)        # a DeclarationUsage: position, normal, tangent,
                                    #   binormal, colour, texture coordinate, blend weight
  usage_index  : int (8-bit)        # which of several elements with the same usage

ENUM DeclarationType, DeclarationMethod, DeclarationUsage
CONSTANT declaration_terminator = { stream 0xFF, offset 0, type "unused", 0, 0, 0 }
CONSTANT max_declaration_length = 64        # including the terminator

# 5. Flexible vertex format bit flags — an older way of describing the same thing,
#    kept because some shipped data still uses it

# 6. Resource usage flags: render target, depth-stencil, write-only, dynamic,
#    auto-generated mip chain, and five more

# 7. Colour helpers: pack four bytes into one 32-bit value in alpha-red-green-blue
#    order, and the transform-state and texture-coordinate-generation identifiers
```

**Invariants**

- **Every numeric value here is frozen.** The engine's material parser maps names from
  shipped text files onto these values, and the renderer backends map these values onto
  their own device's equivalents. Changing one changes the meaning of shipped data.
- The gaps in the numbering are not errors. They are states that the original interface
  retired; the engine never names them, but the surrounding values must keep their numbers.
- A vertex declaration is a list terminated by a sentinel element, not a counted array. The
  sentinel's stream number (255) is what marks the end, and a declaration may hold at most
  64 elements including it.
- The four-character-code formats encode their identity in their value: the bytes of a
  four-letter name packed little-endian into a 32-bit value. That is how they appear in
  texture file headers, so the encoding is frozen by the texture format too.
- Colour values pack alpha in the high byte and blue in the low — the byte order a
  little-endian machine reads as blue-green-red-alpha in memory. This is the convention
  the vertex layouts in
  [`OGF_GContainer_Vertices.hpp`](OGF_GContainer_Vertices.hpp.md) pack into.

## Notes

There is no code in this file. Every declaration is a name for a number, which makes it the
cheapest kind of file to reproduce and the most expensive to get wrong: a single transposed
value produces geometry that renders with the wrong blend mode and no diagnostic at all.

One small piece of build plumbing sits at the top, normalizing how an integer constant with
a platform-specific width suffix is written. It exists because the two compiler families
disagree on the width of that suffix and some of these constants are shifted; it carries no
decision.

The file duplicates declarations that the real interface's headers also provide, and is
included only when they are not present. That conditional inclusion is what lets one
renderer source tree compile on Windows against the real headers and elsewhere against
this. A rebuild adopting this vocabulary as its own should define it unconditionally and
translate at the device boundary instead — the dual-source arrangement means the two can
silently drift.
