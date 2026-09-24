# src/utils/xrLC_Light/R_light.h

> The frozen on-disk record of one baked light, written by the level compiler and read back by the renderer.

**Needs** — [`xrCore/_vector3d.h`](../../xrCore/_vector3d.h.md) · [`xrCore/_math.h`](../../xrCore/_math.h.md) · [Data: Level data](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T1: the record is read by pointing a struct at a mapped file region, so its byte layout, field order and padding are the format.

## Purpose

This header is a **file format**, not a type. It is the only surviving piece of the
offline level-lighting compiler that still ships, and it survives because the renderer
reads the compiler's output: a level's baked light list is stored as a bare array of these
records inside a chunk, with no count and no per-record header, and the reader recovers the
count by dividing the chunk's length by the record's size.

That division is why every byte of the layout is load-bearing. Add a field, reorder two,
or let the compiler pad differently from the reader, and the reader silently produces the
wrong number of lights from the same bytes.

It lives under the lighting compiler's directory rather than under the renderer's because
the compiler is the writer, and the writer owns the format. The compiler itself is not in
this repository — only this record of what it emits. That is the honest summary of this
whole directory.

## State

```text
RECORD BakedLight                  # written as a flat array; size is the format
  type          : int (16-bit)     # 0 directional, 1 point, 2 secondary bounce
  level         : int (16-bit)     # which global-illumination bounce produced it
  diffuse       : real x 3         # colour and intensity together, unclamped
  position      : real x 3         # world space
  direction     : real x 3         # world space
  range         : real             # beyond this the light contributes nothing
  range_squared : real             # precomputed; must equal range * range
  falloff       : real             # precomputed so the curve reaches zero exactly at range
  attenuation0  : real             # constant term
  attenuation1  : real             # linear term
  attenuation2  : real             # quadratic term
  energy        : real             # radiosity only; meaningless at run time
  emitter       : real x 3 x 3     # the triangle this light was emitted from
```

**Invariants**

- **The record size is the format.** The reader divides the chunk length by it and asserts
  the division is exact. A rebuild parsing field by field must still consume exactly this
  many bytes per record.
- **The three light kinds are numbered 0, 1 and 2**, and only the point kind is used by the
  reader that still exists — directional and bounce lights are written by the compiler and
  skipped on load. They remain in the file because the format is shared with the compiler's
  own intermediate passes.
- **`range_squared` and `falloff` are redundant with `range`** and are stored anyway. They
  are precomputations the compiler did once, over a few hundred thousand light samples,
  rather than the renderer doing them per light per frame. A rebuild may recompute them and
  must then still *write* them, because the reader's record size includes them.
- **`falloff` is defined by the requirement that the attenuation curve evaluate to exactly
  zero at the range**, not by any independent physical meaning. It exists so a light can be
  cut off at its range without a visible seam where it is truncated.
- **`diffuse` carries intensity in its magnitude**, so it is not a colour in the
  zero-to-one sense and must not be clamped on load.
- **`energy` is meaningful only inside the radiosity solve** that produced the file. The
  renderer never reads it. It is in the record because the compiler's passes shared one
  record shape with its output.
- **The emitter triangle is initialized to a degenerate one** — a point at the origin and
  two neighbours displaced by the smallest significant epsilon along two axes. It is
  degenerate on purpose: a light that was not emitted by a surface still needs a triangle
  that will not produce a division by zero if something takes its normal. The epsilon is
  the project's own "smallest length that is not zero", not a tuned value.

**Notes**

- The type numbering coincides with the numbering an older graphics interface used for its
  own light kinds, and the renderer's load path still compares against that interface's
  constant rather than the one declared here. The two agree numerically; the coupling is
  accidental and a rebuild should compare against this file's own names.
- Nothing in this repository writes the record. The level compiler that does is a separate
  tool, not shipped here, so a rebuilder who needs to *produce* baked lighting has to write
  one — and this page is the only specification of what it must emit.
