# src/xrEngine/perlin.cpp

> Classic gradient noise in one, two and three dimensions — the source of the weather system's wind gusts, flicker and drift.

**Needs** — [`perlin.h`](perlin.h.md)
**Used by** — [`perlin.h`](perlin.h.md)
**Tier floor** — T2: array indexing and interpolation. It reads the process-wide random generator during setup, which is the only thing tying it to anything.

## Purpose

Several things in the engine need a signal that wanders smoothly but unpredictably: a
lamp's flicker, the sway of foliage, the drift of a weather parameter between its authored
keyframes. This is that signal — coherent noise, meaning nearby inputs give nearby outputs
and distant inputs are uncorrelated.

The algorithm is the standard gradient-noise construction and is reproduced here rather
than derived. What a rebuilder must preserve is not the code but the table size, the
interpolation curve and the octave rule, because outputs are baked into how the shipped
weather configuration was tuned.

## State

```text
RECORD NoiseGenerator
  seed        : int
  ready       : bool                  # tables built on first sample, not at construction
  permutation : list<int>             # 2*256 + 2 entries
  gradients   : list<vector>          # 2*256 + 2 entries; 1, 2 or 3 components
  octaves     : int    (default 2)
  frequency   : real   (default 1)
  amplitude   : real   (default 1)
  octave_phase: list<real>            # one per octave; one-dimensional continuous mode only
  last_time   : real                  # one-dimensional continuous mode only
```

Invariants:

- The tables are **256 entries, duplicated, plus two**. The duplication is what makes the
  index arithmetic wrap without a modulo: an index derived from two adjacent cells can
  exceed 256 and still land on a valid, correctly-wrapped entry. The extra two cover the
  `+1` of the upper cell at the very end. A rebuild that masks indices instead may use 256
  entries flat and get identical results.
- Gradients in two and three dimensions are **normalised to unit length**; in one dimension
  they are not, because a one-dimensional "gradient" normalised to unit length would only
  ever be +1 or -1.
- Tables are built on the first sample, not at construction. That is a laziness the
  original chose so that constructing a generator is free; it also means the seed may be
  changed after construction and before first use.

## Building the tables

```text
FUNCTION initialise(generator)
  seed the process random generator with generator.seed
  FOR i IN 0 .. 255
    permutation[i] = i
    gradient[i] = a vector whose components are uniform in [-1, 1)
    normalise gradient[i]                 # 2D and 3D only
  FOR i FROM 255 DOWN TO 1                # shuffle the permutation
    swap permutation[i] with permutation[random in 0..255]
  FOR i IN 0 .. 257                       # duplicate, so indices may run past the end
    permutation[256 + i] = permutation[i]
    gradient[256 + i]    = gradient[i]
```

**Notes** — the components are drawn as `(random mod 512) - 256` divided by 256, which
gives a value in [-1, 1) quantised to 1/256. The quantisation is not load-bearing; the
range is.

This seeds and draws from the *process-wide* random generator. That is a real coupling and
a hazard: constructing a noise generator mid-run perturbs every other consumer of that
generator, and the noise tables depend on the generator's implementation. A rebuild should
give each generator its own stream — but then its output will differ from the original's,
which matters only if a shipped effect was tuned against specific noise.

## Sampling one octave

```text
FUNCTION noise(generator, point) -> real
  IF not generator.ready THEN initialise(generator)

  FOR EACH axis a
    t       = point[a] + 4096            # bias, so negative coordinates index correctly
    low[a]  = floor(t) masked to 0..255
    high[a] = (low[a] + 1) masked to 0..255
    frac_lo[a] = t - floor(t)            # in [0, 1)
    frac_hi[a] = frac_lo[a] - 1          # in [-1, 0)
    ease[a]    = frac_lo[a]^2 * (3 - 2 * frac_lo[a])

  # hash the cell corners through the permutation table, one axis at a time
  # each corner's contribution is the dot product of its gradient with the
  # offset from that corner to the sample point
  RETURN multilinear interpolation of the corner contributions, using ease[] as the weights
```

**Invariants** — the bias added before flooring is 4096. It exists because the index is
computed by truncation, which rounds towards zero rather than down, so negative coordinates
would fold onto the wrong cell and produce a visible discontinuity at the origin. Any bias
larger than the coordinates ever reached works; 4096 is the original's choice and bounds
the usable coordinate range, since a coordinate near or past it loses precision in the
fractional part. **The source names a "2^N" of 12 alongside it and never uses it**, so the
intended relationship (4096 = 2^12) is recorded but unenforced.

The easing curve is `3t² - 2t³`, the cubic with zero first derivative at both ends. It is
what makes the field look smooth across cell boundaries; the original Perlin choice, and
different from the later quintic that also zeroes the second derivative. Changing it
changes every noise-driven effect subtly.

The corner hashing composes the permutation table across axes — the hash of the x index
selects a row, the y index is added to it, and so on — so that a 256-entry table yields an
effectively unrepeating lattice in two and three dimensions.

## `Get`

**Contract** — sums `octaves` samples, each at twice the frequency and half the amplitude
of the last, starting from the configured frequency and amplitude. Pure apart from the
first-call table build. Returns an unbounded real: with the default two octaves and unit
amplitude the practical range is roughly ±1.5, and callers that need a bounded value must
clamp or scale.

```text
FUNCTION get(generator, point) -> real
  point = point * generator.frequency
  amp   = generator.amplitude
  total = 0
  REPEAT generator.octaves TIMES
    total = total + noise(generator, point) * amp
    point = point * 2
    amp   = amp * 0.5
  RETURN total
```

**Notes** — the doubling and halving are hard-coded, so lacunarity and persistence are not
parameters. A rebuild may expose them; the shipped configuration assumes 2 and 0.5.

## `GetContinious` (one dimension only)

**Contract** — the same octave sum, but driven by *elapsed time* rather than an absolute
coordinate. The caller passes a running clock; the generator takes the difference from the
previous call and advances a separate phase accumulator per octave.

```text
FUNCTION get_continuous(generator, now) -> real
  delta = now - generator.last_time      # first call: delta = now, see below
  generator.last_time = now
  delta = delta * generator.frequency
  amp   = generator.amplitude
  total = 0
  FOR i IN 0 .. generator.octaves - 1
    generator.octave_phase[i] = generator.octave_phase[i] + delta
    total = total + noise(generator, generator.octave_phase[i]) * amp
    delta = delta * 2
    amp   = amp * 0.5
  RETURN total
```

**Invariants** — the octave phase list must have been sized to the octave count; setting
the octave count is what sizes it, and setting the count through the combined parameter
setter does *not*, so a caller that uses the combined setter and then calls this reads past
the end. That is a live hazard in the original, not a design.

**Notes** — the point of the per-octave phase is that each octave advances at its own rate,
so the octaves do not stay in lockstep and the sum does not repeat. This is the sampler the
weather system uses, because it wants "how much has the wind changed since last frame",
not "what is the wind at time t".

On the very first call the previous time is zero and the whole clock value is taken as the
delta, producing one large initial jump. Callers prime it by discarding the first sample.
