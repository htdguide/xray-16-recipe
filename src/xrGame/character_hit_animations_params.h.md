# src/xrGame/character_hit_animations_params.h

> The seven numbers that tune how hard and how often a character flinches.

**Needs** — [`character_hit_animations.cpp`](character_hit_animations.cpp.md)
**Used by** — [`character_hit_animations.cpp`](character_hit_animations.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md)
**Tier floor** — T3: a record of constants

## Purpose

Holds the hit-reaction tuning apart from the code that uses it, so that a debug build can
expose the values to the console and an animator can tune the reactions with the game
running. See [`character_hit_animations.cpp`](character_hit_animations.cpp.md) for what each
value does.

The separation is worth keeping in a rebuild only if the live-tuning affordance is kept. The
values are otherwise constants.

## State

```text
RECORD hit_animation_global_params
  power_factor               = 2.0
  rotational_power_factor    = 3.0
  side_sensitivity_threshold = 0.2
  anim_channel_factor        = 3.0
  block_blend                = 0.5
  reduce_blend               = 0.8
  reduce_power_factor        = 0.5
```

**Invariants** — `block_blend < reduce_blend`, both in the unit interval: they are the two
boundaries of a three-way progress test, and swapping them makes the middle band empty.
`reduce_power_factor` in the unit interval, since it only ever reduces.

**Notes** — none of these seven is derived from anything. They are the values that made the
reactions look right, arrived at with the live-tuning path this file exists to enable.

## the live-tuning globals

**Contract** — one shared instance of the record, edited by console commands, and one flag
saying whether it should be honoured. The values are copied into the set the reaction code
reads only when the flag is on, and only at the moment a model's reactions are bound. A
rebuild that ships the constants directly needs neither.
