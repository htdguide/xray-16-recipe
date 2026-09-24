# src/Layers/xrRenderPC_R4/r4_rendertarget_build_textures.cpp

> The lookup tables the renderer computes for itself at start-up rather than shipping as data: the four material response curves, the jitter tables every sampling kernel reads, and the host-readable image screenshots are copied into.

**Needs** — [`r4_rendertarget.h`](r4_rendertarget.h.md) · [`../xrRender_R2/r2_types.h`](../xrRender_R2/r2_types.h.md) · [`../xrRenderDX11/dx11Texture.cpp`](../xrRenderDX11/dx11Texture.cpp.md) · [`../xrRenderDX11/dx11r_screenshot.cpp`](../xrRenderDX11/dx11r_screenshot.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it writes typed pixel data into a host-side buffer at a stride the device dictates, then hands the buffer over as an immutable allocation's initial contents.

## Purpose

Three of the renderer's inputs are not authored: they are **functions**, tabulated once so
that a program can read them with one sample instead of evaluating them per pixel. Their
contents are the whole content of this page — a rebuilder who can regenerate these three
tables can skip everything else here, and a rebuilder who cannot has a renderer that looks
wrong in ways no amount of correct plumbing fixes.

All three are registered under names in the engine's virtual texture namespace, so the
material system binds them by name exactly like a texture loaded from an archive. Nothing
downstream knows they were computed.

## `build_textures`

**Contract** — creates the screenshot staging image, the material lookup volume, the five
jitter tables and the mipped jitter table; registers each under its engine-side name;
returns nothing. Called once per device lifetime, off the critical path. Allocates
substantially. Uses the global random source, so the jitter tables differ from run to run.

## The material lookup table

A three-dimensional table, 128 by 256 by 4, of two 8-bit unsigned channels.

- **First axis** — the cosine between the surface normal and the direction to the light,
  from 0 to 1 across 128 entries. The diffuse response is nearly linear in this, so it does
  not need resolution.
- **Second axis** — the cosine between the normal and the half vector, from 0 to 1 across
  256 entries. The specular response is a high power of this and needs twice the resolution
  to avoid banding on a highlight.
- **Third axis** — the material identifier, four entries. This is the axis that makes the
  table worth having: the lighting program is identical for every surface in the game, and
  the *only* thing a surface's material changes is which of these four slices it reads.
- **First channel** — the diffuse response. **Second channel** — the specular response.

The two inputs are not used raw. The specular coordinate is first multiplied by the diffuse
coordinate raised to the 1/32 power, and an epsilon is added to it. That product is the
load-bearing preprocessing: it forces the specular term to zero as the surface turns away
from the light, so a highlight cannot survive on a face the light no longer reaches. The
1/32 power makes that falloff almost a step — the specular term is unattenuated until the
surface is nearly edge-on to the light, and then collapses.

Writing the two coordinates after that preprocessing as `d` and `s`, the four rows are:

```text
FUNCTION material_response(row, d, s) -> (diffuse, specular)
  IF row = 0            # a rough, retroreflective surface — cloth, concrete, skin
      diffuse  := d ^ 0.75
      specular := (s ^ 16) * 0.5
  ELSE IF row = 1       # a general glossy surface — the default
      diffuse  := d ^ 0.90
      specular := s ^ 24
  ELSE IF row = 2       # a smooth, plastic surface
      diffuse  := d
      specular := (s * 1.01) ^ 128
  ELSE IF row = 3       # metal: an anisotropic, banded highlight
      # three offset ridges; the surface is brightest where d and s agree,
      # and the sinusoids break that agreement into streaks along the highlight
      a := abs(1 - abs(0.05 * sin(33 * d) + d - s))
      b := abs(1 - abs(0.05 * cos(33 * d * s) + d - s))
      c := abs(1 - abs(d - s))
      diffuse  := d
      specular := (max(a, b, c) ^ 24) * (d ^ (1/7))
  ELSE
      diffuse := 0 ; specular := 0
```

**Invariants**

- **The three lit rows are ordered by increasing smoothness**, and the exponents move
  together: as the diffuse exponent rises toward 1 (less retroreflection) the specular
  exponent rises from 16 to 128 (a tighter highlight). Row 0 additionally halves its
  specular, because a rough surface scatters its highlight rather than reflecting it.
- **The metal row is not a physical model.** It is three near-identical ridge functions
  whose arguments are perturbed by sinusoids at frequency 33, combined by taking the
  brightest. The effect is a highlight broken into streaks — the visual signature of a
  brushed metal — and the frequency, the perturbation amplitude of 0.05 and the final
  `d ^ (1/7)` shoulder are tuning with no derivation. Reproduce them or the game's metal
  stops looking like the game's metal.
- **The corner entry at maximum diffuse and maximum specular is forced to full white in
  both channels.** The preprocessing above would otherwise leave it just short, and that
  corner is what a perfectly-aligned light and view samples — the brightest pixel the engine
  can produce. Clamping it makes that pixel exactly saturated rather than nearly so.

The table is created immutable, so its contents must be complete before the allocation
exists. That is a device constraint, but it reflects a real property: nothing ever rewrites
it.

## The jitter tables

Five tables of 64 by 64, and a mipped copy of the first.

Four of them hold **four signed 8-bit channels**, and each entry packs *two* two-dimensional
sample offsets. The generator, per entry, draws pairs of points in a 256-by-256 integer
square and keeps a point only if its Manhattan distance from every point already kept is at
least 32:

```text
FUNCTION generate_offsets(count) -> list of packed pairs
  kept := empty
  WHILE size of kept < count * 2
      candidate := two independent integers in [0, 256)
      IF for every p IN kept: |candidate.x - p.x| + |candidate.y - p.y| >= 32
          append candidate to kept
  # each output entry packs two of the kept points, with the second point's
  # components swapped relative to the first
  FOR EACH i IN 0 .. count-1
      emit (kept[2i].x, kept[2i].y, kept[2i+1].y, kept[2i+1].x)
```

**Invariants** — the minimum-distance rejection is the point of the whole table. Shadow and
occlusion kernels take a handful of taps per pixel and rotate them per pixel by this table;
if the offsets within one entry clustered, that pixel's kernel would degenerate and show as
a bright or dark speck. A minimum separation of 32 out of 256 guarantees the taps spread
over the kernel no matter which entry a pixel lands on. The rejection loop is unbounded and
relies on the separation being loose enough that it terminates — with at most eight points
in the square it comfortably does.

The tables are **signed**, so an entry read in a program is an offset in both directions
around the pixel, not a displacement away from it.

The **fifth** table holds four 32-bit floats and serves the horizon-based occlusion kernel,
which needs something different: a random *rotation* and a random *radius fraction*, not a
point set.

```text
FUNCTION generate_horizon_jitter(directions) -> (cos, sin, distance, 0)
  # `directions` is how many horizon directions the kernel will march:
  # 4 at the lowest quality setting, 6 at the next, 8 above that
  angle    := uniform in [0, 1) * 2*pi / directions
  distance := uniform in [0, 1)
  RETURN (cos(angle), sin(angle), distance, 0)
```

**Invariants** — the angle is confined to *one* sector, `2π / directions`, not the full
circle. The kernel marches in `directions` evenly spaced directions and adds this rotation
to all of them; confining it to a single sector means the rotated set still covers the
circle evenly. Randomizing over the full circle would let two marched directions collapse
onto each other. The radius fraction randomizes where along the ray the first sample lands,
which trades banding for noise.

The **mipped** copy is built from the first table with **point filtering**, not the usual
box filter. A mip level of a jitter table is read when the kernel wants a coarser set of
offsets; averaging neighbouring entries would pull every offset toward the centre of the
table and destroy exactly the separation the generator worked to guarantee. Point-picking
one of the four keeps a real offset.

## Notes

**The tables are regenerated from a fresh random source on every run**, which means two runs
of the same scene do not produce bit-identical frames. That is accepted: the sampling noise
is meant to be noise, and nothing in the engine compares frames.

**The screenshot staging image is created here** rather than with the other targets because
it is not a target — it is host-readable, screen-sized, and the destination of a copy rather
than of a draw. Its being created alongside the lookup tables is arbitrary and a rebuild may
move it.

**Unrecovered**: the quality-setting-to-direction-count mapping treats the two highest
occlusion settings identically (both march eight directions), so one of the four settings
changes nothing about this table. Whether the higher setting was meant to differ here or
only in the kernel's tap count is not discoverable from this file.
