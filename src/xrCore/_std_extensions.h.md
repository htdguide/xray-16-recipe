# src/xrCore/_std_extensions.h

> The engine's replacement standard library for scalars and raw text: the float validity test, the branch-free minimum and maximum, the bounded string operations, and the compile-time name hash.

**Needs** — [`_std_extensions.cpp`](_std_extensions.cpp.md) · [`crc32.cpp`](crc32.cpp.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md)
**Used by** — [`pch.hpp`](../utils/xrMiscMath/pch.hpp.md) · [`Motion.hpp`](Animation/Motion.hpp.md) · [`FileSystem.cpp`](FileSystem.cpp.md) · [`_compressed_normal.cpp`](_compressed_normal.cpp.md) · [`_fbox.h`](_fbox.h.md) · [`_fbox2.h`](_fbox2.h.md) · [`_math.cpp`](_math.cpp.md) · [`_matrix.h`](_matrix.h.md) · [`_obb.h`](_obb.h.md) · [`_rect.h`](_rect.h.md) · [`_std_extensions.cpp`](_std_extensions.cpp.md) · [`_stl_extensions.h`](_stl_extensions.h.md) · [`_vector2.h`](_vector2.h.md) · [`stdafx.h`](stdafx.h.md) · _and 6 more_
**Tier floor** — T1: it classifies floating-point values by their bit pattern and performs bounded copies into fixed-size caller buffers.

## Purpose

The engine decided which small operations it wanted and where their edge cases sit, rather than taking what the language offered. Three of those decisions are load-bearing and the rest is convenience.

## `_valid` — the float validity test

**Contract** — Reports whether a floating-point value is usable. Returns false for: signaling and quiet not-a-number, both infinities, and **both subnormals**. Returns true for normal values and both zeroes.

**Invariants** — **Subnormals are rejected.** That is the decision. A subnormal is a finite, representable number, and most validity tests accept it; this one does not, because a subnormal appearing in a position, a velocity or a normal means an underflow cascade has already happened and the physics solver will produce garbage from it a few steps later. Rejecting early turns a mysterious explosion into an assertion at the source.

**Notes** — This predicate is asserted on vectors, matrices, quaternions and spheres throughout the physics and animation layers in debug builds, and it is one of the main reasons those layers are debuggable at all. A rebuild should keep both the predicate and the liberal assertion of it.

## Branch-free minimum, maximum and absolute value

**Contract** — For each signed integer width, a minimum, maximum and absolute value computed with arithmetic and shifts instead of a comparison; for unsigned widths, absolute value is the identity. Generic comparison-based forms exist for everything else.

**Notes** — The source itself flags these as "magic specializations that really require profiling to see if they are worth this effort", and on a modern target they almost certainly are not — compilers emit conditional moves for the obvious form. A rebuild should write the obvious form and measure before doing anything else. What must survive is only that a minimum, maximum and absolute value exist for every width used.

## Bounded string operations

**Contract** — Copy, append and format into a caller buffer of stated size, with a size-deducing form for fixed-size arrays so the size is never passed wrongly. Each truncates rather than overrunning.

**Invariants** — Two different implementations exist and the build chooses between them: a checking one that fails loudly on truncation, and a shipping one that truncates silently. This is the same policy as the assertion classes — **diagnose in development, degrade in shipping** — and it means a rebuild must decide which behaviour is the contract. The engine's answer is: silent truncation is acceptable in a player's session, and unacceptable while developing.

## `strext`

**Contract** — Points at the **last** period in a path, or nothing. This is the engine's file-extension rule: everything after the final period, including the period. A path with no period has no extension; a path whose directory contains a period and whose file does not will return the directory's period, which the callers avoid by splitting the path first.

## `MemFill32`

**Contract** — Fills a region with a repeating 32-bit pattern. The count is in **32-bit units, not bytes** — a mismatch here overruns by a factor of four and the name does not say so.

## `strhash` and the literal suffix

**Contract** — A compile-time hash of a string, and a literal suffix that applies it. Used so that a name can be a switch label — the console's command dispatch and several factory tables are written this way.

```text
FUNCTION strhash(text) -> int (32-bit, wraps)
  # A multiply-by-33-and-add hash. Note the initial value is 5385, not the
  # 5381 of the algorithm this is taken from -- an off-by-four that is now
  # frozen by every switch statement written against it.
  h = 5385
  FOR EACH ch IN text
    h = (h * 33) + ch
  RETURN h
```

**Invariants** — The seed is 5385. The algorithm it derives from uses 5381. A rebuild that "fixes" the seed changes every hashed dispatch value; since the values are only ever compared against each other within one build, fixing it is safe *provided* nothing serialized one — and nothing does. Reproducing it exactly is therefore optional, and knowing it is wrong is not.

## Checksum declarations

**Contract** — The checksum functions defined in [`crc32.cpp`](crc32.cpp.md) are declared here, which is why almost every file has access to them.

## `timestamp`

**Contract** — Declared here, defined in [`_std_extensions.cpp`](_std_extensions.cpp.md).
