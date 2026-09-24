# src/xrPhysics/params.cpp

> Reads the global scale on collision damage out of the configuration, once, and pre-squares it.

**Needs** — [`params.h`](params.h.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md) · [Data: configuration](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`params.h`](params.h.md)
**Tier floor** — T3: reads one number from a text file.

## Purpose

One tunable that the shipped game data owns rather than the console. It scales how much damage a
physical object does when it hits something — the difference between a thrown crate being a nuisance
and being lethal.

## State

```text
object_damage_factor : real     # default 1.0; replaced at load with the square of the configured value
```

## `load_params`

**Contract** — reads `object_damage_factor` from the `physics` section of the shipped configuration
and stores its **square**. Does nothing at all if the configuration is not mounted yet, leaving the
default of 1.0 in place; the caller is expected to invoke it once the virtual filesystem and
configuration are up. Reads one key, allocates nothing, and must not be called concurrently with a
step.

```text
FUNCTION load_params()
  IF settings NOT mounted THEN RETURN          # start-up order, not an error
  f = settings.read_real("physics", "object_damage_factor")
  object_damage_factor = f * f
```

**Notes** — the squaring is the only decision on the page, and it is not arbitrary. Collision damage
is derived from the *energy* of a non-elastic collision, which goes as the square of the relative
approach speed (see [`MathUtilsOde.h`](MathUtilsOde.h.md) and
[`collisiondamagereceiver.cpp`](collisiondamagereceiver.cpp.md)). Squaring the configured number
here makes the value an author types a multiplier on the *speed* scale rather than on the energy
scale: doubling it makes an object as dangerous as one moving twice as fast, which is the intuition
a designer has. A rebuild that applies the factor to a linear damage quantity must not square it.

If the key is missing the read fails hard rather than defaulting — the shipped data always has it,
and a mod that removes it is a data error the engine surfaces immediately rather than silently
running with different balance.
