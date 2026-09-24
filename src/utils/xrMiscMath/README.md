# src/utils/xrMiscMath — the numeric layer

Chapter 3 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order).

## What this module is responsible for

Every spatial quantity in the engine — a position, a direction, a bone pose, a camera, a
contact normal, a particle's frame — is one of three records: a 3-vector, a 4×4 transform,
or a rotation quaternion. Those records are *declared* in the core module's headers, where
the short operations are written inline. This module holds everything else: the bodies of
the operations that were too long to inline, plus the engine's entire treatment of angles
as points on a circle, plus one numerically careful normalization that exists because the
ordinary one cannot represent its own input.

It is the smallest module in the build by file count and one of the largest by reach.
Nothing above it can avoid it, and almost every line of it runs thousands of times per
frame.

## Where it sits

It rests on nothing but the platform prologue, the elementary numeric helpers from
[chapter 2](../../xrCommon/README.md) and the type declarations from the core module's
math headers. It calls into no subsystem, allocates nothing, holds no state, and logs
nothing. Everything from chapter 4 upward depends on it.

### Why this is a separate module

The engine's core is a shared library and this module is a static one, and every other
module — including the core itself — links it. The effect is that the math bodies are
compiled *into* each consumer rather than called across the shared-library boundary. For
code that runs a few thousand times per frame that is the difference between a call the
compiler can inline and one it cannot.

The cost is a circular-looking arrangement: the declarations live in the core's headers,
the definitions live here, and the core links this module to get them. The source carries
a maintainer's note that the whole module should be dissolved. **A rebuild should treat
this as one module with the core's math headers and let its own build system decide where
the code lands** — the decision that must survive is "this arithmetic is not reached
through a dynamic-linkage boundary", not the file layout that achieves it.

## The ideas the twins assume

### Transform conventions

Stated once here; the twins are terse about them.

- A transform is sixteen floats in **row-major order**, `m11 m12 m13 m14 m21 …`, with
  **translation in the fourth row** and the fourth column `(0, 0, 0, 1)` for everything
  that is not a projection.
- Points are **rows**, not columns: a point is transformed by multiplying it on the *left*
  of the matrix.
- Consequently the composition routine takes its arguments **in the order they are
  written, not the order they apply**: composing `(A, B)` yields a transform that applies
  **B first and A second**.
- The first three rows are named after what they mean — right, up (normal), forward
  (direction) — and the fourth is the translation.
- **A positive angle rotates clockwise** when looking along the axis. This is the opposite
  of the usual mathematical convention and it is baked into every authored angle in the
  game data.
- Euler triples are **heading, pitch, bank**, composed **Z then X then Y**. Heading is
  measured about the vertical axis from forward, increasing toward the left; pitch lifts
  out of the horizontal plane; bank rolls about forward.
- The world is Y-up and, for most game-layer purposes, horizontal: the vector type carries
  first-class "distance ignoring height" operations for that reason.

### The epsilon ladder

Four different smallness tests are in play and they are not interchangeable:

| name | value | used for |
|---|---|---|
| tightest | `1e-7` | "has this angle arrived at its target" |
| general | `1e-5` | "are these two reals the same", default similarity |
| loose | `1e-3` | geometric coincidence tests in the layers above |
| smallest normal | the float format's own floor | "can this magnitude be squared at all" |

The last is not an epsilon and must not be replaced by one. It is the boundary between a
vector whose direction can be recovered the ordinary way and one whose direction can only
be recovered by rescaling first.

### The floating-point environment is configured elsewhere, and it differs by target

Every thread the engine starts is put into **flush-to-zero and denormals-are-zero** mode on
the x86 family — and, because the instruction to do so does not exist there, is *not* on
ARM and PowerPC. So the same source produces subtly different results on different
targets: on x86 a subnormal intermediate becomes exactly zero, on ARM it does not. This
interacts directly with the normalization guards in this module and with the
component-flushing helper.

A rebuild must make a deliberate choice here rather than inherit one. The honest reading of
the original is that the mode was set for speed, the divergence was not intended, and
nothing in the game is known to depend on it — but "not known to" is the strongest claim
the source supports.

### "4-wide" is a layout, not an instruction set

This module contains no wide-register code. The 4-wide shape is in the *data*: a transform
row is four floats, the 4-vector is four floats, and the 3-vector is deliberately three so
that it packs into vertex streams without padding. The arithmetic is written one component
at a time and vectorization is left to the compiler. A rebuild should express the
operations in terms of 4-wide float lanes and let its target choose, exactly as
[the platform assumptions](../../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)
describe — but it should not assume there is a hand-vectorized original to match.

## The numeric contract

This is what a rebuilder needs to decide before writing a line.

### Precision

**Single precision is the working width everywhere**, with exactly two exceptions, both in
this module: the robust normalization does all its arithmetic in double and narrows only
on the final store, and the reciprocal-square-root helper it calls is double in and double
out. Double appears nowhere else in the engine outside the rigid-body solver.

A rebuild that promotes everything to double will be *more* accurate and still wrong, in
two ways. First, the on-disk and on-wire formats are single-precision bit patterns
(see below) and every value that reaches them must round to the same float the original
produced. Second, several thresholds in this module — the gimbal-lock guard in particular
— are scaled to single-precision rounding, and in double they become so tight that the
degenerate branch never fires when it should.

### What is an approximation, and what is not

The obvious suspects are **not** approximations here, and a rebuilder should not
"restore" speed that was never traded away:

- **Reciprocal square root is exact.** The helper named for the classic fast
  approximation computes an honest `1 / sqrt`. Its only caller needs the accuracy. Do not
  substitute an estimate.
- **Sine and cosine are the platform library's**, at full precision. There is no lookup
  table anywhere in this module. Neither is there one for the arctangents used by the
  Euler-angle extraction.
- **Square root and the general normalizations are exact**, up to the rounding of the
  working width.

Approximations do exist in the engine's numeric layer, but they live in the headers this
module implements rather than here, and each is opt-in at the call site:

- A **polynomial arc-sine** (and the arc-cosine built on it), valid only for non-negative
  arguments, in the bit-twiddling header and again privately inside the quaternion type,
  where the spherical interpolation uses it. It is a seventh-order odd polynomial and it
  is visibly wrong near the ends of its range. Animation blending therefore has a
  systematic angular error, and reproducing the original's *look* means reproducing that
  error, not removing it.

Where genuine imprecision is baked in deliberately, it is in *distributions*, not in
arithmetic:

- The random-direction draw is **not** uniform on the sphere — it is denser near the poles.
  The uniform version sits in the source, commented out. Particle sprays and creature
  wander directions were tuned against the biased one.
- The cone-limited random direction is only approximately a cone of the requested
  half-angle, and the random-point-in-a-ball draw clusters toward the centre.

These are tuning inputs. A rebuild that corrects them changes how the game looks and moves.

### Angle normalization is exact, and its interval is a contract

Angles are plain reals in radians, normalized by an explicit truncate-and-repair rather
than by a remainder operation, so the result is exact rather than accumulated. Two
intervals are in use and must both be carried: unsigned `[0, 2π]` for stored headings and
signed `[-π, π]` for differences and for anything fed to interpolation. One detail is
worth reproducing knowingly: the fast paths test their bounds inclusively, so an input of
exactly a full turn comes back as a full turn rather than zero.

### Where exactness actually matters

Most of this module's output is consumed, used for a frame and discarded, and small
differences are invisible. Three paths are different, and a rebuild must hold them to the
bit:

1. **Anything that reaches the graphics device.** Transforms are uploaded as constant
   buffers and vectors as vertex-stream components, both as raw single-precision bytes in
   the layout described above. The byte order and the value must both match; there is no
   conversion step to hide a difference in.
2. **Anything that is serialized.** Positions and orientations go into save files and into
   the authored spawn records, as single-precision values. A rebuild that computes an
   orientation in double and narrows on write will not in general produce the same bits as
   one that computed it in single — and a save that round-trips through the engine's own
   Euler conversion must come back to the same triple, which is why the gimbal-lock
   threshold and the heading convention are contract rather than taste. The save format is
   [frozen only against itself](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence),
   so a rebuild may redefine it — but only if it gives up loading existing saves.
3. **Anything that crosses the network.** Positions and angles are quantized into
   bit-packed fields by the transport layer, and the server and every client must quantize
   the *same* value to the *same* bucket or prediction and reconciliation diverge. The
   quantization lives in the network module, but what it quantizes is produced here, so
   two ends running differently-rounded angle arithmetic will drift apart even with an
   identical protocol.

Everything else — lighting, particle motion, camera smoothing, AI steering — tolerates
whatever the target's rounding gives it.

### Failure behaviour

This module never reports an error to a caller in the ordinary sense and never logs. Its
three responses to a degenerate input are, in order of how often they are used:

- **assert and continue** — checked in instrumented builds only, absent from the shipped
  one, so in practice the operation proceeds on garbage;
- **leave the value untouched** — the "safe" normalizations and the quaternion extraction's
  last-resort path, which means the caller silently keeps a non-unit value;
- **substitute a defined default** — zero for an undefined heading, straight-up for the
  normalization of a zero vector, zero for the angle between degenerate directions.

The third is the only one a rebuild should keep as-is. The first two are places where the
original chose speed over diagnosis; a rebuild in a tier that can afford a result type
should use one, and will find bugs.

## The files

| twin | role |
|---|---|
| [`vector.cpp`](vector.cpp.md) | Angle arithmetic on the circle; the out-of-line body of the 3-vector — normalization, blending, heading/pitch, random directions, basis construction |
| [`matrix.cpp`](matrix.cpp.md) | The 4×4 transform: composition, affine and full inversion, the rotation constructors, Euler conversion, the six axis permutations |
| [`quaternion.cpp`](quaternion.cpp.md) | Extracting a rotation quaternion from a transform, with the four-route fallback that keeps the division well conditioned |
| [`vector3d_ext.cpp`](vector3d_ext.cpp.md) | The value-returning face of the vector operations: products, angle between directions, ground-plane rotation |
| [`xrMiscMath.cpp`](xrMiscMath.cpp.md) | The robust normalization for vectors too short to square in single precision |
| [`pch.hpp`](pch.hpp.md) | Shared prologue; carries the one decision that this module compiles the definitions everyone else imports |
| [`pch.cpp`](pch.cpp.md) | A compilation unit for the prologue; no content |

The directory also holds a build script and two project files for one toolchain. They
describe the module as a static library compiled against the source root, and nothing in
them survives into a rebuild.
