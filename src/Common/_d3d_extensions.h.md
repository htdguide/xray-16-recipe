# src/Common/_d3d_extensions.h

> The engine's own light and surface-material records, laid out to be byte-identical to the fixed-function graphics structures they replaced.

**Needs** — [`xrCore/FixedVector.h`](../xrCore/FixedVector.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: both records assert an exact byte size, because they were once handed to a driver directly.

## Purpose

Two records that the engine passes around internally — a light source and a surface
material — and which have a frozen size for a historical reason: the original engine handed
them straight to the fixed-function graphics pipeline, so their layout had to match that
interface exactly. The pipeline is long gone and both records are now purely the engine's
own, but the sizes are asserted at build time and the field order preserved.

A rebuild keeps the *fields* and drops the size assertions. They are named here because
anyone reading the original will find the assertions and wonder what enforces them.

## State

```text
RECORD Light                          # exactly 104 bytes
  kind         : ENUM { point = 1, spot = 2, directional = 3 }   # 32-bit
  diffuse      : colour (four real)
  specular     : colour (four real)
  ambient      : colour (four real)
  position     : three real           # world space
  direction    : three real           # world space, unit
  range        : real                 # beyond this distance the light contributes nothing
  falloff      : real                 # shape of the spot cone's edge
  attenuation  : three real           # constant, linear and quadratic terms
  inner_angle  : real                 # spot cone: full intensity within
  outer_angle  : real                 # spot cone: zero intensity beyond

RECORD Material                       # exactly 68 bytes
  diffuse   : colour (four real)
  ambient   : colour (four real)      # only the colour channels are meaningful
  specular  : colour (four real)
  emissive  : colour (four real)
  power     : real                    # specular exponent: higher is a tighter highlight
```

**Invariants**

- The light kind is a 32-bit value and must be, because it is the first field of a
  size-asserted record.
- The attenuation triple is evaluated as `1 / (a + b·d + c·d²)` at distance `d` — the
  classic fixed-function falloff. Only the engine's oldest render path still uses it; the
  modern paths compute their own falloff and read only position, direction, colour and
  range.
- A light's range is the culling bound, not a shading parameter: the visibility pass uses
  it to decide which lights touch which sectors.

## `Light.set`

**Contract** — initialize a light of a given kind pointing at and positioned at one world point.
Zeroes every field first, sets diffuse and specular to full white, leaves ambient black,
normalizes the direction (tolerating a zero vector), and sets the range to the square root
of the largest representable real.

**Notes** — the default range is effectively infinite and is deliberately *not* the largest
representable value: the square root leaves headroom so that squaring a distance against it
does not overflow. That is the one non-obvious constant in the file and it is the kind of
detail that is invisible until a distance comparison produces a non-number.

Position and direction are set from the same point, which is only meaningful for a point
light; a spot or directional light must have its direction set afterwards.

## `Light.scale`

**Contract** — multiply the three colour terms' colour channels by a brightness factor, leaving
alpha alone. Used by the weather system to dim a level's authored lights with the time of
day.

## `Material.set`

**Contract** — initialize a material with a colour, with or without an explicit alpha, zeroing
everything else. Diffuse and ambient are set to the same colour; specular and emissive stay
black and the specular exponent stays zero, which means no highlight.

**Notes** — three spellings of the same operation exist, differing only in whether alpha is
given and whether the colour arrives as components or as a colour value. That is a
convenience, not a decision.

## Notes

Both records can be compiled out by name, which is how a translation unit that includes the
real graphics headers avoids a clash. That is a build mechanic with no counterpart in a
rebuild.

The size assertions have two forms: against the real graphics structures when those headers
are present, and against literal byte counts otherwise. The literal form is what runs on
every platform but one. Keeping both means the numbers stay honest — if the real structure
ever changed, one build configuration would catch it.
