# src/xrGame/stalker_animation_offsets.cpp

> Loads and serves the per-animation aim correction that keeps a stalker's weapon pointing where the AI thinks it points.

**Needs** — [`stalker_animation_offsets.hpp`](stalker_animation_offsets.hpp.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md)
**Used by** — [`stalker_animation_offsets.hpp`](stalker_animation_offsets.hpp.md)
**Tier floor** — T3: parse two numbers per line and look them up

## Purpose

Aiming is computed by the AI as a yaw and pitch for the head and torso, but the *visible*
weapon direction during an animation is whatever the animator authored. Where the two
disagree — a lean, a crouch, a cover pose — a per-animation correction closes the gap. This
file owns the table of those corrections.

## State

```text
RECORD AnimationOffsets
  entries : map<text, Rotation>   # animation identifier -> (yaw, pitch, roll=0)
```

**Invariants** — roll is always zero; only two of the three angles are authored. Angles are
stored in radians although the configuration states them in degrees, so the conversion
happens exactly once, at load.

## `load`

**Contract** — reads every key/value line of one configuration section. The key is the
animation identifier; the value is a comma-separated pair of angles in degrees, yaw then
pitch. Converts both to radians and inserts. Duplicated keys are whatever the
configuration reader yields; nothing here deduplicates. Allocates; does not block.

```text
FUNCTION load(section_name)
  FOR EACH (key, value) IN config_section(section_name)
    yaw   := degrees_to_radians(number(field(value, 0)))
    pitch := degrees_to_radians(number(field(value, 1)))
    entries[key] := Rotation(yaw, pitch, 0)
```

**Notes** — a missing or unparsable field yields zero rather than an error. Given that a
zero offset is also the "no correction" answer, a malformed line is indistinguishable from
an absent one, which is why nothing validates this section.

## `offsets`

**Contract** — returns the correction for one animation identifier, or a zero rotation when
the identifier is absent. Never fails; returns by value, so the caller may hold it across
a reload. Pure.
