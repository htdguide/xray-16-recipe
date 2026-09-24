# src/xrParticles/noise.cpp

> Classic three-dimensional gradient noise over a 256-entry table, and the fractal sum built
> on it that drives the turbulence action.

**Needs** — [`noise.h`](noise.h.md)
**Used by** — reached through its declarations in [`noise.h`](noise.h.md); callers name that, not this file.
**Tier floor** — T2: a table of gradients and float interpolation. It sits in the frame budget,
so a tier with boxed floats would hurt, but nothing here needs explicit layout.

## Purpose

One coherent-noise field, sampled per particle per step by the turbulence action. Everything
about it is standard — the version from *Texturing and Modeling* — and the only decisions a
rebuild must copy are the ones that change the *field itself*: the table size, the way the
table is built, and the lacunarity of the fractal sum. Two effects that use turbulence will
not look the same if any of those differ.

The file is separate because nothing else in the chapter needs it, and because it owns process
lifetime state that the rest of the chapter does not.

## State

```text
# Built once, never changed, shared by every effect on every thread. Read-only after
# initialization, so no synchronization is needed once it is built.
permutation : list<int>   # 514 entries: a shuffle of 0..255, then the first 258 repeated
gradients   : list<vec3>  # 514 entries, same repetition: unit vectors, uniform on the sphere
```

The repetition at the end is the standard wrap trick: an index and its neighbour can both be
read without a modulo, because the table is long enough that the neighbour of the last entry
is a copy of the first.

## `noise3Init`

**Contract** — builds both tables. Must run before any sampling; runs exactly once, lazily, on
the first turbulence step. Not thread-safe, and not made so: the first step of the first
turbulent effect happens to be serial. Seeds and consumes the platform's integer random
generator.

```text
FUNCTION noise3Init()
  seed the platform integer generator with 1        # the table must be identical every run
  FOR i IN 0..255
    REPEAT                                          # rejection-sample the unit ball
      v = three draws, each uniform on -1..1 in steps of 1/256
    UNTIL |v|² <= 1
    gradients[i] = v / |v|                          # uniform direction, no corner bias
  FOR i IN 0..255 : permutation[i] = i
  shuffle permutation                               # see the note
  FOR i IN 0..257
    permutation[256+i] = permutation[i]
    gradients[256+i]   = gradients[i]
```

**Invariants** — the generator is seeded with a fixed value so the field is the same on every
run and every machine. That is the only reason this produces reproducible visuals.

**Notes** — two things here look wrong and are frozen anyway, because the tables they produce
*are* the field:

- The shuffle walks the table backwards in steps of two, so it swaps only half the entries,
  and it starts one entry past the end of the range it just filled — reading a slot that is
  still zero and writing it back into the table. The result is a valid-but-lopsided
  permutation, not the Fisher–Yates shuffle the shape of the loop suggests. Reproduce it
  literally or the noise pattern changes.
- The gradient draw quantizes each component to 1/256 before rejection, so the directions come
  from a finite set. Harmless, and part of the same reproducibility.

The bigger consequence of this routine is one nothing declares: it *reseeds the platform's
integer generator* to a fixed value as a side effect, and that generator is the sign source for
the module's normal-distribution sampler
([`particle_core.cpp`](particle_core.cpp.md)). So the first step of the first turbulent effect
in a session silently resets part of the particle system's randomness. A rebuild should give
this table its own generator; doing so is a visible improvement and a deliberate divergence
from the original's number stream.

## `noise3(point)`

**Contract** — one sample of gradient noise. Pure, allocation-free, no failure path. Range is
roughly −1…1; the field is zero at lattice points and smooth between them, with a period of
256 along each axis.

```text
FUNCTION noise3(p) -> real
  FOR EACH axis
    # Bias by 10000 before flooring: the cheap float-to-int on this path truncates
    # toward zero rather than flooring, so a negative coordinate would fold the
    # lattice back on itself. The bias is large enough to cover any coordinate a
    # level can have, and is a whole number so it does not shift the lattice.
    t      = p[axis] + 10000
    cell   = truncate(t)
    lo     = cell AND 255 ; hi = (lo + 1) AND 255
    frac   = t - cell     ; frac_minus_one = frac - 1
    smooth = frac² · (3 - 2·frac)              # the ease curve: zero slope at both ends

  hash the eight corner indices through the permutation table
  FOR EACH of the eight corners
    contribution = gradient[corner] . offset_from_that_corner
  trilinearly interpolate the eight contributions with the three smoothed fractions
  RETURN 1.5 · result        # scales the gradient-noise range out to about -1..1
```

## `fractalsum3(point, frequency, octaves)`

**Contract** — a signed fractal sum. Pure. Cost is linear in the octave count, and the
turbulence action calls it four times per particle per step, which makes this the chapter's
hottest routine by a wide margin.

```text
FUNCTION fractalsum3(p, frequency, octaves) -> real
  base = frequency ; f = frequency ; sum = 0
  REPEAT octaves TIMES
    sum = sum + noise3(p · f) / f      # amplitude falls as 1/frequency: pink-ish
    f   = f · 2.059                    # see the note
  RETURN sum · base                    # restores the authored frequency as an amplitude scale
```

**Notes** — the octave ratio is 2.059 rather than 2. An exact doubling makes every octave share
the same lattice alignment, so their zero crossings pile up and the field shows a visible
grid; an irrational-looking ratio near 2 decorrelates them. The precise value is not special
beyond being near 2 and not rational-looking, but it is part of the field: change it and
shipped turbulence effects swirl differently.

The trailing multiply by the *initial* frequency means the authored frequency controls
amplitude as well as scale. That coupling is almost certainly unintended and is load-bearing
anyway — every authored turbulence magnitude in the shipped data was tuned against it.

## `turbulence3(point, frequency, octaves)`

**Contract** — identical to the fractal sum but accumulating each octave's absolute value,
which creases the field at its zero crossings and makes it non-negative. Pure. Dead: nothing
in the engine calls it. A rebuild may omit it, at the cost of an authoring option that was
never wired up.
