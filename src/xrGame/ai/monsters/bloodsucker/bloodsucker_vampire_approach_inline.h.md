# src/xrGame/ai/monsters/bloodsucker/bloodsucker_vampire_approach_inline.h

> Close the distance to feed: run flat out at the victim's navigation cell, ignoring cover entirely, and keep re-aiming at it as he moves.

**Needs** — [`bloodsucker_vampire_approach.h`](bloodsucker_vampire_approach.h.md) · [`bloodsucker.h`](bloodsucker.h.md) · [`state.h`](../state.h.md)
**Used by** — [`bloodsucker_vampire_approach.h`](bloodsucker_vampire_approach.h.md)
**Tier floor** — T3: one movement request and a path target re-issued each update

## Purpose

The first node of the vampire tree. Everything about it is a decision to drop the creature's usual caution for the duration: no cover in the route, no time-minimising compromise, aggressive acceleration with braking disabled, and a route rebuilt at the creature's attack cadence.

The caution is not missing by oversight — the creature is *already* partly invisible for the whole of the enclosing tree, so the cloak is the cover, and any further hiding would only slow the approach.

## State

Stateless.

## `VampireApproachState`

**Contract** — on entry, prime the path builder. Each update: request a run, select the aggressive acceleration profile with braking off, resolve the enemy's current navigation cell, aim the path at that cell's centre, set the route rebuild interval from the creature's configured attack cadence, forbid cover in the route, ask to stop essentially on top of the target, and vocalise aggression.

**Invariants** — the target is recomputed every update, so the route follows a moving victim. Nothing here ends the state; the enclosing tree's selector does.

```text
FUNCTION execute()
  request_action(run)
  acceleration(aggressive, braking = false)

  target_vertex = navigation cell the enemy stands in
  path.target        = (centre of target_vertex, target_vertex)
  path.rebuild_every = section.attack_rebuild_interval
  path.use_covers    = false
  path.stop_distance = 0.1
  vocalise(aggressive)
```

**Notes** — the creature is routed to the *centre of the enemy's cell* rather than to the enemy's exact position. That snapping is what makes a 0.1-unit stop distance meaningful: the route ends at a fixed point in the world rather than at a moving one, so the path builder can report arrival instead of chasing a target that moves faster than the route rebuilds.

The rebuild interval comes from the creature's configuration section rather than being fixed here, which is the general rule in this chapter — the shape of the behaviour is code, its cadence is data.
