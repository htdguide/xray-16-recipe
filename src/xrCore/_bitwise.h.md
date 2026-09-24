# src/xrCore/_bitwise.h

> Integer-level operations on floating-point numbers, power-of-two arithmetic, population counts, and two polynomial approximations the animation and collision code lean on.

**Needs** — [`xr_types.h`](xr_types.h.md) · [`math_constants.h`](math_constants.h.md)
**Used by** — [`vector.cpp`](../utils/xrMiscMath/vector.cpp.md) · [`math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md) · [`_color.h`](_color.h.md) · [`vector.h`](vector.h.md)
**Tier floor** — T1: it reads and writes the bits of a float directly, and several routines depend on the exact 32-bit binary format.

## Purpose

A grab bag, but a coherent one: everything here exists because a naive expression of the same thing was measured to be too slow on the hardware of the era. The sign manipulations avoid a branch, the population counts avoid a loop, the approximations avoid a library call in an inner loop. A rebuild on modern hardware should start from the *naive* expression and only reach for these if a measurement says so — with one exception, noted below, where the approximation is load-bearing because it changes results.

## The float bit layout

The file names the masks and patterns of the 32-bit binary floating-point format: the sign bit, the mantissa field, the exponent field, the absolute-value mask, and the bit patterns of one, a half, two, the largest finite value, and a not-a-number. A rebuild needs these only if it keeps the bit-level routines; the layout itself is fixed by the platform assumption in §4 of the system requirements.

## Sign predicates and manipulation

**Contract** — ask whether a number is negative or positive, and force it to one sign or the other, in place.

```text
negative(f)      -> the sign bit is set          # true for negative zero
positive(f)      -> the sign bit is clear
set_negative(f)     set the sign bit
set_positive(f)     clear the sign bit
```

**Invariants** — these are *bit* operations, so negative zero counts as negative and forcing the sign of a not-a-number leaves a not-a-number. On architectures where reading a float's bits through an integer is unsafe or slow, the file falls back to the standard sign predicates and to negating the absolute value, which differ only in their treatment of negative zero.

**Notes** — the payoff is in [`_fbox.h`](_fbox.h.md)'s transform, which uses `negative` twelve times in a row to decide which of two accumulators each transformed axis contributes to. Written as a comparison that is twelve branches; written as a sign-bit test it is twelve predictable ones. That is the measured case, and it is why the routine exists.

## Power-of-two helpers

**Contract** — isolate the lowest set bit, test whether a value is a power of two, and round a value up to the next power of two. All defined for both signed and unsigned inputs, all total, all undefined for zero in the rounding case (it loops forever).

```text
lowest_set_bit(v)   -> v AND (two's complement negation of v)
is_power_of_two(v)  -> lowest_set_bit(v) == v
round_up_pow2(v)    -> start at lowest_set_bit(v), shift left until >= v
```

**Notes** — the lowest-set-bit trick relies on two's complement representation, which the platform assumptions guarantee. The rounding loop is linear in the bit position; a rebuild should use its language's bit-scan.

## Population count

**Contract** — count the set bits of an 8-, 32- or 64-bit value.

**Notes** — three implementations: a two-step fold for the byte, a three-way parallel fold using two magic masks for the word, and two word counts summed for the long word. Where the compiler offers an intrinsic it is used instead, which is the honest answer everywhere now. The magic masks partition the word into groups of three bits and then of nine, which is why they look arbitrary; a rebuild should call its language's bit count and delete all of it.

## Rounding to integer

**Contract** — floor and ceiling of a real, returned as a machine integer. Total for finite inputs in range; out-of-range inputs are undefined.

**Notes** — these were once hand-written bit manipulations and are now the standard library calls. The names survive because they are used in a few hundred places, and the *rounding* they perform is load-bearing: the quantizations in [`NET_utils.cpp`](NET_utils.cpp.md) and the colour conversions in [`_color.h`](_color.h.md) are specified in terms of floor, and substituting truncation or round-to-nearest changes bytes on the wire.

## Validity predicates

**Contract** — ask whether a number is denormal, and whether its exponent is in a suspicious range.

```text
is_denormal(f)   -> the exponent field is entirely zero
is_gremlin(f)    -> the exponent byte, shifted down and offset by 0x20,
                    exceeds 0xC0
```

**Notes** — the second predicate's name and threshold are not explained anywhere in the source. It flags very large magnitudes and not-a-numbers; the offset and the bound look like a hand-tuned "this number is almost certainly garbage" test used while chasing a specific bug. Nothing in the shipped code calls it. A rebuild should use the standard finite-number test and drop this.

Denormal detection matters because the engine *flushes denormals to zero* per thread (see [`_math.cpp`](_math.cpp.md)); a value that would be denormal is therefore zero by the time anything sees it, on any thread the engine started correctly.

## `apx_asin` and `apx_acos`

**Contract** — approximate arcsine and arccosine for arguments in `[0, 1]` only. Outside that range the result is meaningless — there is no clamping and no check.

```text
FUNCTION apx_asin(x)              # x in [0, 1]
  x2 <- x * x
  RETURN x * (0.892399 + x2 * (1.693204 + x2 * (-3.853735 + x2 * 2.838933)))

FUNCTION apx_acos(x)
  RETURN (pi / 2) - apx_asin(x)
```

**Invariants** — this is a degree-seven odd polynomial fit, not a truncated series; the coefficients are a minimax-style fit over the unit interval and are not derivable from anything. Copy them exactly.

**Notes** — this is the one approximation in the file that is **load-bearing rather than optional**. The same polynomial is duplicated inside [`_quaternion.h`](_quaternion.h.md) and is what the spherical interpolation uses to find the angle between two rotations. Every blended animation in the game is evaluated through it. Substituting the exact arccosine produces slightly different blend weights, which is visible as a different pose at the same blend factor — small, but it means an animation-comparison test against the original will not match. A rebuild that wants byte-comparable animation output must use this polynomial; one that only wants plausible animation may use the exact function.

The approximation's error grows toward the ends of the interval and is worst near 1, which is exactly where the interpolation uses it most (two nearly-identical rotations). The interpolation guards that case separately by degenerating to a straight line, which is why the error never shows.
