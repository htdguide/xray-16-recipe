# src/xrCore/_compressed_normal.cpp

> Packs a unit vector into sixteen bits and unpacks it again — the encoding shipped vertex data and network packets use for normals.

**Needs** — [`_compressed_normal.h`](_compressed_normal.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md) · [Data and persistence](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — [`_compressed_normal.h`](_compressed_normal.h.md)

**Tier floor** — T1: a sixteen-bit field layout that appears in shipped model data and on the wire, plus a table indexed by the raw word.

## Purpose

A direction costs twelve bytes as three floats and two bytes in this encoding. That saving
multiplies across every vertex of every mesh and every normal sent over the network, so the
engine stores directions packed wherever the consumer can afford the decode. The encoding
is **frozen**: it appears inside shipped model files, so a rebuild must reproduce both
directions bit for bit, not merely "some 16-bit normal encoding".

The scheme is an octahedral-style projection. Take the absolute value of the direction and
project it onto the plane `x + y + z = 126`; every direction in the positive octant lands
in a triangle with integer corners `(0,0)`, `(126,0)`, `(0,126)`. Two of the three plane
coordinates determine the third, so only two are stored. The three signs are stored
separately, which is what recovers the other seven octants.

## The bit layout

```text
RECORD PackedNormal : int (16-bit, little-endian on disk)
  bit 15      : sign of x        # set means negative
  bit 14      : sign of y
  bit 13      : sign of z
  bits 12..7  : x plane coordinate, 6 bits, range 0..63
  bits 6..0   : y plane coordinate, 7 bits, range 0..126
```

**Invariants**

- The three sign bits are the top three; the low thirteen bits are the magnitude index.
  Masking the sign bits off yields a value in `0 .. 8191`, which is exactly the index space
  of the lookup table below. That equality is the reason the field widths are 6 and 7 and
  not, say, 7 and 6.
- The x field is only six bits wide because of the *folding* step described below: the
  triangle is folded into a rectangle so that x never needs more than 64 values.
- The all-zero word is the encoding of the zero vector on the way in, and decodes to
  `(0, 0, 1)` on the way out. The round trip is therefore **not** identity for a zero
  input. Nothing checks; callers are expected never to store a zero normal.

## The normalization table

**Contract** — 8192 entries of one real each, built once at startup and read-only
thereafter. Entry `i` holds the reciprocal length of the plane point that the thirteen-bit
magnitude index `i` decodes to. Reading it during decode replaces a square root with a
load.

```text
FUNCTION build_table() -> list<real>
  FOR index FROM 0 TO 8191
    x = index >> 7
    y = index AND 0x7f
    IF x + y >= 127 THEN               # the folded half: unfold it
      x = 127 - x
      y = 127 - y
    z = 126 - x - y
    table[index] = 1 / sqrt(x*x + y*y + z*z)
  RETURN table
```

**Notes** — The table is 32 KB and is built by
[`pvInitializeStatics`](_compressed_normal.h.md), which the engine's startup calls from
[`_math.cpp`](_math.cpp.md). A rebuild may skip the table entirely and compute the
reciprocal length inline; that is one square root per decoded normal and is a straight
speed-for-memory trade, not a correctness question — the *values* are identical either
way.

Note that the table is indexed by the **folded** thirteen bits and performs the unfold
itself. That means the same unfold appears twice, here and in the decoder, and the two must
agree or the lengths come out wrong for half the sphere.

## `pvCompress`

**Contract** — Takes a direction of any length, returns the sixteen-bit word. Pure,
allocation-free, total: an invalid (non-finite) input is rejected and yields zero, and an
all-zero input yields zero. The input need *not* be unit length — the projection divides
the length out — which is why nothing normalizes first.

```text
FUNCTION compress(v) -> int (16-bit)
  IF v is not finite THEN RETURN 0
  IF v.x, v.y and v.z are all within epsilon of zero THEN RETURN 0

  word = 0
  # Record each sign, then work in the positive octant only.
  IF v.x is negative THEN word = word OR 0x8000 ; v.x = |v.x|
  IF v.y is negative THEN word = word OR 0x4000 ; v.y = |v.y|
  IF v.z is negative THEN word = word OR 0x2000 ; v.z = |v.z|

  # Project onto the plane x+y+z = 126. This is a projective map: the
  # length of v is divided out, so an unnormalized input is fine and a
  # zero input would divide by zero -- which is why it was handled above.
  w = 126 / (v.x + v.y + v.z)
  x = floor(v.x * w)                   # 0..126
  y = floor(v.y * w)                   # 0..126, with x + y <= 126

  # Fold the triangle into a rectangle so x fits in six bits. The upper
  # half of the triangle maps onto the lower half reflected; the decoder
  # recognizes the folded half by x + y >= 127 and undoes it.
  IF x >= 64 THEN
    x = 127 - x
    y = 127 - y

  RETURN word OR (x << 7) OR y
```

**Invariants** — After the projection, `0 <= x <= 126`, `0 <= y <= 126` and
`x + y <= 126`. After the fold, `0 <= x <= 63` and `0 <= y <= 127`. The fold's inverse is
detected by `x + y >= 127`, which is unreachable for an unfolded point precisely because
the unfolded sum is at most 126 — the constant 126 and the test against 127 are the same
decision seen twice, and changing one without the other silently corrupts half the sphere.

**Notes** — Signs are taken by flipping the sign bit of the float rather than by
arithmetic. That is an optimization, not a decision: a rebuild uses absolute value and
comparison, and must only be careful that negative zero counts as *positive* here, since
the sign test used is "is the sign bit set" and a negative zero would therefore set a sign
bit for a component that is zero. The decoded value is unaffected because the component
decodes to zero either way.

The quantization is uniform on the projection plane, not on the sphere, so angular error is
worst near the octant corners. Roughly one part in 126 per axis — about half a degree at
worst. That is acceptable for shading normals and is *not* acceptable for anything that
integrates the direction over time, which is why physics never stores a direction this way.

## `pvDecompress`

**Contract** — Takes the sixteen-bit word, writes a unit direction. Pure,
allocation-free, total — every one of the 65536 words decodes to something. Requires the
table to have been built; reading it uninitialized yields zeros and therefore a zero
vector, silently.

```text
FUNCTION decompress(word) -> (real, real, real)
  x = (word AND 0x1f80) >> 7
  y = word AND 0x007f

  adjust = table[word AND 0x1fff]      # look up BEFORE unfolding: the
                                       # table is indexed by the stored bits

  IF x + y >= 127 THEN                 # the folded half
    x = 127 - x
    y = 127 - y

  v = (adjust * x, adjust * y, adjust * (126 - x - y))

  IF word has bit 15 THEN v.x = -v.x
  IF word has bit 14 THEN v.y = -v.y
  IF word has bit 13 THEN v.z = -v.z
  RETURN v
```

**Invariants** — The table lookup uses the *stored* thirteen bits, before the unfold; the
component reconstruction uses the *unfolded* pair. Both are correct because the table
applies the same unfold internally, but a rebuild that unfolds first and then indexes the
table gets the wrong scale for every folded direction.

The result is unit length to within the table's precision. Nothing renormalizes.

**Notes** — The third component is `126 - x - y` and is never stored: that is the whole
point of projecting onto the plane. It is always non-negative after the unfold, so the sign
bit is the only thing that can make z negative.
