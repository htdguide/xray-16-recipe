# src/Layers/xrRenderGL/glr_constants_cache.h

> The writing half of the constant model: how a named value reaches every stage that declared it, and why this backend's "cache" caches nothing.

**Needs** — [`glr_constants.cpp`](glr_constants.cpp.md) · [`xrRender/r_constants.h`](../xrRender/r_constants.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glDetailManager_VS.cpp`](glDetailManager_VS.cpp.md) · [`glr_constants.cpp`](glr_constants.cpp.md)
**Tier floor** — T1: every setter is a direct driver call on the per-draw path.

## Purpose

The shared renderer's command list owns an object called the constant cache, whose contract is: accept named values during a pass, coalesce them, and push them to the device once before the draw. The Direct3D 11 backend implements it that way — values land in a shadow copy of a constant buffer and one upload flushes the lot.

This backend implements the same interface with **no cache at all**. Every set writes straight through to the driver, and the flush is empty. The name is inherited, and a reader who trusts it will mis-model the frame cost badly: a pass that sets twenty constants makes twenty driver calls here and one buffer upload on the other backend.

Stating that plainly is the point of this page. The original marks the buffer-backed version as unfinished, and a rebuild targeting any modern API should implement it rather than copy this.

## State

`Stateless.` — the object holds nothing; it is a namespace for the setters, which reach through the constant's own recorded locations.

## `set(constant, value)` — the fan-out

**Contract** — writes one named value to *every stage that declared the name*. This is the core of the by-name model: the caller says "set `m_shadow` to this matrix" and does not know or care which stages use it.

```text
FUNCTION set(constant, value) -> ()
  IF constant targets the pixel stage    THEN write(constant.pixel_load,    value)
  IF constant targets the vertex stage   THEN write(constant.vertex_load,   value)
  IF constant targets the geometry stage THEN write(constant.geometry_load, value)
  IF constant targets the whole program  THEN write(constant.program_load,  value)
```

The overloads accept a 4x4 matrix, a 4-vector, four scalars, a single real, and a single integer. Each checks the constant's declared value type against what it is being handed and faults on a mismatch in a checked build.

## `write(load, matrix)` (private)

**Contract** — writes a matrix, transposed, in the shape the shader declared.

```text
FUNCTION write(load, M) -> ()
  SELECT load.class
    2x4 -> send the first two columns of M as rows
    3x4 -> send the first three columns of M as rows
    4x4 -> send all four columns of M as rows
    otherwise -> FAIL WITH "constant's run-time shape does not match its use"
```

**Invariants** — **the transpose is mandatory and it is done twice over.** The engine stores matrices in the row-major convention the old API used; the shaders are written expecting the column-major multiplication order of the new one. The code transposes explicitly when packing the rows *and* asks the driver to transpose again on upload. The two cancel for the square case and combine correctly for the non-square ones, where the engine's 3-row transform must arrive as a 4-column-by-3-row matrix. A rebuild that changes the engine's matrix storage must revisit every shader's multiplication order at the same time; they are one decision, not two.

Writing a 3x3 or smaller shape is a hard failure, matching the parse side.

## `write(load, vector)` · `write(load, x, y, z, w)` · `write(load, real)` · `write(load, int)` (private)

**Contract** — write a 2-, 3- or 4-component value, or a scalar, at the recorded location. The shape comes from the *constant's* declared class, not from what the caller passed, so a caller handing four scalars to a 2-vector constant writes only the first two — which is how the same call site serves shaders that declare different widths for the same name.

**Notes** — every write comes in two forms, chosen by the same separable-program capability the rest of the backend branches on: one that names the program object explicitly, and one that writes to whichever program is currently bound. The first is what makes a separable pipeline workable — a value can be pushed into the vertex program without disturbing the bound pipeline — and it is why the constant's `Load` carries a program handle at all.

## `set_array_element(constant, element, value)`

**Contract** — writes one element of an array constant, by adding the element index to the recorded location.

**Invariants** — this relies on an array's uniform locations being *consecutive from the base*, which the API guarantees for a single array. It does not hold across arrays, and it does not hold if the driver strips unused elements — which it may. The detail-object renderer ([`glDetailManager_VS.cpp`](glDetailManager_VS.cpp.md)) writes batches of instance transforms this way, four elements per instance, and is the only heavy user.

The destination selection inside this path takes the *last* matching stage rather than fanning out to all of them, so an array declared in both the vertex and pixel stages would only be written in one. No shipped shader does that; it is a latent difference from the scalar path worth knowing about.

## `flush`

**Contract** — does nothing. See Purpose.
