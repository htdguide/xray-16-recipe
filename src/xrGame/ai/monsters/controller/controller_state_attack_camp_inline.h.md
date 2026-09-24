# src/xrGame/ai/monsters/controller/controller_state_attack_camp_inline.h

> The camping sub-state: on entry, find how wide an arc the creature's cover actually affords by tracing against geometry, then sweep the gaze between those limits at random intervals.

**Needs** — [`controller_state_attack_camp.h`](controller_state_attack_camp.h.md) · [`controller_animation.h`](controller_animation.h.md) · [`controller_direction.h`](controller_direction.h.md) · [`../state.h`](../state.h.md) · [`../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md) · [`../../../xrAICore/Navigation/ai_object_location.h`](../../../../xrAICore/Navigation/ai_object_location.h.md) · [Seam: Static collision database](../../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`controller_state_attack_camp.h`](controller_state_attack_camp.h.md)
**Tier floor** — T2: a pair of discrete raycast sweeps on entry, then a cheap per-tick update

## Purpose

What a controller does while it waits: it stands in cover and watches. The state is
interesting for one reason — it does not assume a fixed field of view. On entry it asks the
navigation mesh which direction its current cell is *least* exposed from, then traces
outward from that direction in both senses until it hits geometry, and takes those two hit
directions as the arc it can actually watch. The creature then sweeps its gaze between the
two limits at randomized intervals.

So the same state produces a wide, lazy sweep in the open and a tight watch down a corridor,
without any authoring.

## State

See [`controller_state_attack_camp.h`](controller_state_attack_camp.h.md).

## Tuning

Five constants, all in this file:

| Constant | Value | Meaning |
|---|---|---|
| sweep half-width | a quarter turn | the widest arc either side of the cover direction |
| trace step | 10 degrees | the angular resolution of the geometry sweep |
| trace distance | 3 world units | how far a wall must be to bound the arc |
| reversal interval | uniform in 2000–5000 ms | how long the gaze rests at each limit |
| completion timeout | 2000 ms | how long the state runs before it completes |

## `initialize`

**Contract** — establish the watch arc. Ask the navigation mesh for the direction in which the
creature's current cell has the *highest* low-cover value — that is, the direction it is most
sheltered from, sampled at ten-degree resolution. Set the arc to a quarter turn either side
of it, then narrow each side by tracing outward in ten-degree steps until a static hit inside
three units bounds it. Finally aim the creature's body at a point three units along the cover
direction.

```text
FUNCTION initialize()
  base <- the direction of greatest low cover at my cell, at 10-degree resolution
  angle_from <- base - a quarter turn
  angle_to   <- base + a quarter turn

  FOR ang stepping LEFT from base while within a quarter turn of it
    IF a static ray from my centre along ang hits within 3 units THEN
      angle_from <- ang; BREAK

  FOR ang stepping RIGHT from base while within a quarter turn of it
    IF a static ray from my centre along ang hits within 3 units THEN
      angle_to <- ang; BREAK

  target_angle <- angle_from
  face a point 3 units along base
```

**Notes** — the two loops are what turn a generic quarter-turn arc into one shaped by the room
the creature is actually standing in. A wall three units to the left stops the sweep there, so
the creature does not conspicuously stare into it.

The cover value comes from the level's precomputed per-vertex cover data, so the *initial*
direction costs nothing; only the narrowing costs rays, and it costs at most eighteen of them
per entry into the state.

The three-unit trace distance is doing double duty as both "close enough to be a wall worth
respecting" and, unchanged, as the distance at which the look-at point is placed.

## `execute`

**Contract** — per tick: advance the sweep if its interval has elapsed, then aim the
creature's *gaze* at a point three units along the current target angle, and set its body
state to the sneaking pose with an idle torso.

**Notes** — the aiming goes through the creature's head-and-spine driver, not its body, so a
camping controller sweeps its head while its body stays put. That is the visible difference
between this creature and every other one in the chapter.

## `update_target_angle`

**Contract** — if the reversal interval has elapsed, draw a new one uniformly between two and
five seconds and flip the target between the two limits.

**Notes** — randomizing the dwell is what keeps a room of controllers from sweeping in
lockstep. It is the same device the abilities use for their cooldowns.

## `check_start_conditions` / `check_completion`

**Contract** — the state may begin only when the creature *cannot* currently see its enemy;
it completes when the creature can see the enemy, or after two seconds either way.

**Notes** — the two-second cap means camping is never a resting state: the selector is forced
to re-choose at least that often, so a controller that loses sight of the player watches
briefly and then does something else. Without the cap it would stand and sweep indefinitely.
