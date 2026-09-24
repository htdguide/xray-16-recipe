# src/xrGame/visual_memory_params.cpp

> Loads one vision profile from a configuration section, and decides which of its numbers a monster is allowed to need.

**Needs** — [`visual_memory_params.h`](visual_memory_params.h.md) · [`memory_space.h`](memory_space.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`visual_memory_params.h`](visual_memory_params.h.md)
**Tier floor** — T3: reads named numbers out of the configuration store

## Purpose

The loader for [`visual_memory_params.h`](visual_memory_params.h.md). Trivial except for
one branch, which is the reason the file exists at all: whether a creature that is not a
stalker loads the full accumulator tuning or only the two numbers every owner needs.

## State

`Stateless.`

## `Load`

**Contract** — fills one profile from the named configuration section. Takes a flag saying
whether the owner is *not* a stalker. The two universal numbers are read first, then — if
the build gives monsters stalker-grade vision, or the owner is a stalker — the eight
accumulator numbers. Every one of those eight is mandatory and a missing key is a hard
failure; `still_visible_time` is optional and defaults to zero.

```text
FUNCTION load(section, not_a_stalker)
  transparency_threshold := config.real(section, "transparency_threshold")   # required
  still_visible_time     := config.int(section, "still_visible_time") ELSE 0 # optional

  IF monsters do not use stalker vision AND not_a_stalker
    RETURN                      # the eight below stay at whatever the record held

  min_view_distance       := config.real(section, "min_view_distance")
  max_view_distance       := config.real(section, "max_view_distance")
  visibility_threshold    := config.real(section, "visibility_threshold")
  always_visible_distance := config.real(section, "always_visible_distance")
  time_quant              := config.real(section, "time_quant")
  decrease_value          := config.real(section, "decrease_value")
  velocity_factor         := config.real(section, "velocity_factor")
  luminocity_factor       := config.real(section, "luminocity_factor")
```

**Invariants** — after a full load every field is set. After a short load the eight
accumulator fields are **left at whatever they previously held**, which for a freshly
constructed profile is uninitialized. That is only safe because the same build switch that
takes the short path also makes the accumulator unreachable for that owner. A rebuild
should instead give the record defined defaults and keep the branch as "do not require these
keys", which is the same behaviour without the trap.

**Notes** — `still_visible_time` defaulting to zero is what makes "seen now" and "still
believed in" collapse to the same predicate for any creature whose section omits it. That
is the conservative default: a creature only keeps shooting at a vacated position if its
configuration says how long for.

The build switch that decides whether monsters get the accumulator is enabled in the shipped
configuration, so the short path is dead in practice. It survives because turning it off is
how the original was profiled — the full accumulator on every monster in a level was once
too expensive, and the switch is the fossil of that. Whether it still costs anything on
modern hardware is not recoverable from the source.
