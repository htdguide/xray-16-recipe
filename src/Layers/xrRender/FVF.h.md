# src/Layers/xrRender/FVF.h

> The six vertex layouts the engine's own immediate drawing uses, their byte shapes, and the two conventions that make them work across both backends.

**Needs** — [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`BufferUtils.h`](BufferUtils.h.md) · [`D3DUtils.h`](D3DUtils.h.md) · [`ParticleEffect.cpp`](ParticleEffect.cpp.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_DBG.cpp`](R_Backend_DBG.cpp.md) · [`dxRainRender.cpp`](dxRainRender.cpp.md) · [`dxStatGraphRender.cpp`](dxStatGraphRender.cpp.md) · [`dxThunderboltRender.cpp`](dxThunderboltRender.cpp.md) · [`dxUIRender.cpp`](dxUIRender.cpp.md) · [`r__sector.cpp`](r__sector.cpp.md) · [`r__sector_traversal.cpp`](r__sector_traversal.cpp.md)
**Tier floor** — T1: every record here is an exact byte layout handed to a graphics driver.

## Purpose

Model geometry has its layouts described by the model files. Everything the engine draws *directly* — debug shapes, the user interface, decals, particles — needs a layout named in code, and this is that small closed set. Six of them, each a record plus the packed identifier that describes it to the device.

## The layouts

```text
RECORD PositionColour                    # 16 bytes
  position : vector3
  colour   : int (32-bit, packed)

RECORD PositionTexture                   # 20 bytes
  position : vector3
  texcoord : vector2

RECORD PositionColourTexture             # 24 bytes
  position : vector3
  colour   : int (32-bit, packed)
  texcoord : vector2

RECORD ScreenColourTexture               # 28 bytes
  position : vector4        # ALREADY in screen space; see below
  colour   : int (32-bit, packed)
  texcoord : vector2

RECORD ScreenColourTexture2              # 36 bytes: two texture coordinate pairs
RECORD ScreenColourTexture4              # 52 bytes: four texture coordinate pairs
```

**Invariants**

- Every record is packed to four-byte alignment. The device's stride must match the record exactly; a compiler-chosen padding byte silently misinterprets every vertex after the first.
- The four-coordinate variant's setters only ever write **two** of its four coordinate pairs; the other two are left as whatever the buffer held. Callers that use all four write them directly. This is a trap in the original, not a decision.

## Convention one: the colour byte order differs between backends

```text
FUNCTION pack_colour(c)
  on one backend  -> swap the first and third byte of c
  on the other    -> c unchanged
```

**Invariants** — The engine's colour values are produced in one byte order throughout and one of the two graphics APIs expects the other. Rather than convert at the driver boundary, the conversion is folded into the vertex *setters* — so a vertex written through a setter is correct and a vertex written field by field is not. A rebuild must pick one internal order and convert in exactly one place, which is the right version of this decision; folding it into the setters is how it ended up applied inconsistently. Note that the conversion is applied by only *one* of the position-colour setters and not the others, which is a live inconsistency in the shipped source.

## Convention two: the pre-transformed vertex

**Invariants**

- The three screen-space layouts hold a **four-component position that is already in device coordinates**: x and y in the normalized device range, z as the depth to write, and w as one over the homogeneous divisor. Nothing transforms them. This is the fixed-function transformed-and-lit vertex.
- The convenience setters plant a fixed depth of 0.0001 and a fixed divisor of 0.9999. Those are "as near as possible without being at the clip plane" and "one, but not exactly" — the second is the load-bearing one: a divisor of exactly one is what a genuine w of one would be, and 0.9999 was evidently chosen to keep the vertex distinguishable from a transformed one in a debugger. There is no rendering consequence at the precision involved, and a rebuild writes one and zero.
- The `transform` operation projects a world position into this form by hand: multiply by the view-projection, divide by w, and **negate the y**. The negation is the difference between the device's downward screen axis and the engine's upward one, and it appears here, in the vertex record, rather than in the projection matrix. That placement is why the two-coordinate and four-coordinate variants each carry their own copy of the same eight lines.

**Notes** — A rebuild targeting any current graphics API replaces all three screen-space layouts with one layout plus an orthographic projection, as the chapter-4 README says. What it must preserve is the *pixel-centre convention* the shipped user-interface art was authored against: the engine's screen coordinates address pixel corners, not centres, and getting that wrong blurs every sharp edge in the interface by half a pixel.
