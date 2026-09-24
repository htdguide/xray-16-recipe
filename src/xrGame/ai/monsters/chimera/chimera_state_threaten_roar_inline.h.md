# src/xrGame/ai/monsters/chimera/chimera_state_threaten_roar_inline.h

> Four seconds of standing, facing and bellowing.

**Needs** — [`chimera_state_threaten_roar.h`](chimera_state_threaten_roar.h.md) · [`state.h`](../state.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_direction_base.h`](../control_direction_base.h.md)
**Used by** — [`chimera_state_threaten_roar.h`](chimera_state_threaten_roar.h.md)
**Tier floor** — T3: a leaf state

## Purpose

The display itself. Reached only from [`chimera_state_threaten_inline.h`](chimera_state_threaten_inline.h.md), which nothing instantiates.

## State

Stateless; the entry timestamp is the shared machinery's.

```text
roar_duration = 4000 ms     # fixed in code
turn_duration = 1200 ms     # how long the creature is given to face the target
```

## `execute`

**Contract** — One tick of the display: stand, raise the animation layer's threaten flag, ask the sound layer for the threatening vocalisation, and turn to face the target over a bounded time.

```text
FUNCTION execute()
  action = stand_idle
  raise_animation_special_parameter(threaten)
  state_sound = threatening
  face(enemy, over turn_duration)
```

**Notes** — The threaten flag is raised every tick and consumed by the creature's special-parameter hook. The chimera's hook implements nothing — see [`chimera.cpp`](chimera.cpp.md) — so the flag has no effect and the creature bellows in its idle pose. The flag is the channel by which a behaviour state asks the *animation* layer for a variant, rather than forcing a clip directly; for a creature whose model carries a threaten clip it would play one.

Facing is given an explicit duration rather than a speed, so the turn completes within the display regardless of how far round the target is.

## `check_completion`

**Contract** — True once four seconds have passed since the state was entered. Nothing else ends it; the parent decides what comes next.
