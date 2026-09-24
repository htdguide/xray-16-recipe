# src/Common/OGF_GContainer_Vertices.hpp

> The vertex layouts of compiled static level geometry, the quantization that produces them, and the widening that turns them back into plain floats.

**Needs** — [`d3d9compat.hpp`](d3d9compat.hpp.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: these are byte layouts handed to a graphics driver with explicit strides, and the quantization relies on exact integer ranges.

## Purpose

A level's static geometry is stored in a shared vertex pool, and each vertex in it is
squeezed into 28 or 32 bytes. This file is both halves of that: the *packed* layouts as
they exist on disk and in a vertex buffer, and the arithmetic that converts a full-precision
authored vertex into one and back out again.

The packing is not incidental. A level carries a few million static vertices, and the
difference between a packed vertex and a naive one is the difference between fitting the
level's geometry in a graphics device of the era and not. Every field that could be
narrowed was, and the tricks used to narrow them — stealing a texture-coordinate's low bits
into a colour channel's unused alpha — are the load-bearing content of the file.

## State

```text
# the five packed layouts, all tightly packed with no padding

RECORD LightmappedVertex              # 32 bytes — geometry lit by a baked light map
  position      : three real          # 12 bytes, full precision
  normal        : int (32-bit)        # three bytes of direction + one byte of hemisphere light
  tangent       : int (32-bit)        # three bytes of direction + LOW BYTE of texture u
  binormal      : int (32-bit)        # three bytes of direction + LOW BYTE of texture v
  base_uv       : two int (16-bit)    # HIGH part of the base texture coordinate
  lightmap_uv   : two int (16-bit)    # the light map coordinate

RECORD VertexLitVertex                # 32 bytes — geometry lit per vertex at compile time
  position      : three real
  normal        : int (32-bit)        # direction + hemisphere light
  tangent       : int (32-bit)        # direction + low byte of texture u
  binormal      : int (32-bit)        # direction + low byte of texture v
  color         : int (32-bit)        # baked red, green, blue + sun contribution in alpha
  base_uv       : two int (16-bit)

RECORD InstancedModelVertex           # 32 bytes — a repeated prop placed many times
  position      : three real
  normal        : int (32-bit)
  tangent       : int (32-bit)
  binormal      : int (32-bit)
  misc          : four int (16-bit)   # first two are the texture coordinate;
                                      #   the other two are unread — see notes

RECORD PositionOnlyVertex             # 12 bytes — depth and shadow passes
  position      : three real
```

```text
# the widened forms, produced at load time for devices that cannot use the packed ones

RECORD LightmappedVertexWide          # 32 bytes
  position, normal (as a colour), base_uv as two real, lightmap_uv as two real

RECORD VertexLitVertexWide            # 28 bytes
  position, normal, color, base_uv as two real

RECORD InstancedModelVertexWide       # 28 bytes
  position, normal, color, texture coordinate as two real
```

**Invariants**

- Each packed layout has a matching *declaration* — a list of (byte offset, element type,
  semantic) entries handed to the graphics device so it can interpret the buffer. The
  declaration and the record must agree field for field; they are written next to each
  other precisely so a change to one is visible against the other, and there is no
  automatic check that they still match.
- The declared sizes are 32 bytes for every packed layout except the position-only one, and
  28 for two of the three widened ones. 32 is not a coincidence: it is the cache line
  quarter the geometry pipeline of the era read in.
- Direction vectors (normal, tangent, binormal) are stored as three unsigned bytes mapping
  −1..+1 onto 0..255, with the fourth byte carrying something unrelated. They are therefore
  *not* unit vectors after unpacking and the shader must renormalize.

## Direction packing

**Contract** — map a unit direction into three bytes, with a caller-supplied fourth byte.

```text
FUNCTION pack_direction(v, extra_byte) -> int (32-bit)
  v <- (v + 1) * 127.5        # -1..+1  ->  0..255
  RETURN pack_rgba(clamp(floor(v.x), 0, 255),
                   clamp(floor(v.y), 0, 255),
                   clamp(floor(v.z), 0, 255),
                   extra_byte)
```

**Notes** — the widening step reverses it as `value * 2 - 1` on the normalized 0..1 channel
form, which is the same mapping expressed in the colour domain. The asymmetry — floor on
the way in, exact on the way out — costs up to half a quantization step of bias, which at
8 bits is below the shading's own noise floor.

## Base texture coordinate packing

**Contract** — the base texture coordinate is the one field that genuinely needs more than 16
bits, because level geometry tiles its textures: a coordinate may range over ±32 tile
repeats and still needs sub-texel precision. It is stored as 24 bits, split across two
fields of two different vertex attributes.

```text
FUNCTION pack_base_uv(coordinate) -> (high : int (16-bit, signed), low : int (8-bit))
  MAX_TILES = 32
  QUANT     = 32768 / MAX_TILES         # = 1024 steps per tile repeat
  scaled    <- coordinate * QUANT
  high      <- clamp(floor(scaled), -32768, 32767)
  low       <- clamp(floor(255.5 * (scaled - high)), 0, 255)
  RETURN (high, low)

FUNCTION unpack_base_uv(high, low) -> real
  RETURN (high + low) * (32 / 32768)    # the reconstruction the widening path uses
```

**Invariants**

- The range is ±32 texture repeats. A surface authored with more tiling than that wraps
  and is visibly wrong; the level compiler is expected to split it.
- The high 16 bits live in the vertex's texture-coordinate attribute and the low 8 bits
  live in the **alpha channel of the tangent** (for u) and **of the binormal** (for v).
  Those alpha bytes are otherwise unused because the direction packing only needs three.
  This is the file's central trick and the reason the tangent and binormal must be
  unpacked before the texture coordinate can be.
- The reconstruction adds the low byte to the high value *before* scaling, treating the low
  byte as a fractional extension in the same units. It is only correct because the
  quantization step is exactly 1024 per tile and the low byte spans one step.

## Light map coordinate packing

**Contract** — a light map coordinate is confined to one texture, so it needs only ±1 and is
stored as a plain 16-bit fixed-point value.

```text
FUNCTION pack_lightmap_uv(coordinate) -> int (16-bit, signed)
  RETURN clamp(floor(coordinate * 32768), -32768, 32767)
```

## Hemisphere and sun lighting

**Contract** — the alpha byte of the packed *normal* carries the hemisphere (ambient sky)
contribution at that vertex, baked by the level compiler. For vertex-lit geometry, the
alpha of the packed colour carries the sun contribution. Both are 8-bit quantities in a
channel the geometry did not need.

**Invariants** — this is why the packed normal cannot simply be widened to a 3-vector: its
fourth channel is lighting data, not padding, and the widening path preserves it by keeping
the colour form rather than converting to floats.

## Notes

The instanced-model layout stores four 16-bit values where only two are read: the texture
coordinate. The remaining two are written by the level compiler and consumed by nothing in
this engine. They are most likely a second texture coordinate or a per-instance index from
an earlier design; nothing in the source says, and the bytes must still be written for the
layout's stride to be right.

Half of this file compiles only inside the level compiler — the constructors that take
full-precision authored vertices and produce packed ones. The engine itself only ever goes
the other way. Keeping both directions in one file is what keeps the packing and unpacking
arithmetic visibly paired, and is the best argument for the file existing at all.

The widened forms exist because not every graphics device accepts the packed element types.
Producing them at load time costs memory and load time but nothing per frame; a rebuild
targeting a modern device can read the packed layout directly everywhere and delete the
widening path, at the cost of doing the coordinate reconstruction in the shader.
