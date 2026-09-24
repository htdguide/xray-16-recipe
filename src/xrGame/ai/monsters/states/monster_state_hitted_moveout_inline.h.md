# src/xrGame/ai/monsters/states/monster_state_hitted_moveout_inline.h

> Creep back toward whatever shot you, hopping from one covered spot to the next, walking while far
> off and stalking once close.

**Needs** — [`monster_state_hitted_moveout.h`](monster_state_hitted_moveout.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md)
**Used by** — [`monster_state_hitted_moveout.h`](monster_state_hitted_moveout.h.md)
**Tier floor** — T3: a leg-by-leg destination chooser over the cover system

## Purpose

The inward half of the shot-from-nowhere oscillation, and the more considered of the two. Where the
break-away is a raw retreat, this leaf advances in **legs**: it asks the cover system for a covered
spot near the hit point, walks there, and when that leg is done asks again. The result is an animal
working its way toward a shooter through concealment rather than crossing the open ground in one
run.

## State

```text
RECORD MoveOutState
  target_position : vector
  target_vertex   : optional<int>   # absent = no cover found; head straight for the hit point
```

**Invariant** — the pair is rewritten as a unit whenever a leg ends; the position is meaningless
when the vertex is absent, and the fallback path ignores both.

## `select_target` — choosing a leg

**Contract** — score every cover point within fifteen units of the creature by how well it sits in
the band ten to twenty units from the last hit position, take the best, and record its position and
vertex. On failure clear the vertex.

```text
FUNCTION select_target()
  # the cover system enumerates candidates around the creature and an evaluator ranks them
  evaluator = prefer_a_point_whose_distance_to(hit_memory.last_hit_position)
                lies in [10, 20], with zero tolerated deviation
  best = cover_system.best_cover(around = self.position, radius = 15, evaluator)
  IF best EXISTS  target_position, target_vertex = best.position, best.vertex
  ELSE            target_vertex = absent
```

**Notes** — the two radii are measured from different things and that is the whole design. The
search radius — fifteen units around the **creature** — bounds how long one leg may be. The
evaluator's band — ten to twenty units from the **hit point** — says where a leg should end up:
close enough to make progress, far enough not to walk into the open. Together they produce a
sequence of short hops that each end nearer the shooter.

The deviation argument is zero, meaning the band is treated as exact rather than as a soft
preference. Because the band's lower bound is ten, legs never aim closer than ten units to the
shooter; the final three units of the completion test are reached by drift, or by the fallback
below. All four numbers are hard-coded.

## `execute`

**Contract** — when the current leg's route has been planned since entry and the path builder
reports the route nearly finished, choose a new leg. Then hand the path builder the current
destination — or the hit position itself when no cover was found — and choose the gait from range:
walk while more than ten units from the hit point, creep inside that. Disable the acceleration
profile and braking, and use the idle voice.

```text
FUNCTION execute()
  IF path.route_planned_since(entry_time) AND path.route_nearly_finished(within 1.5)
    select_target()

  IF target_vertex EXISTS  path.target = (target_position, target_vertex)
  ELSE                     path.target = hit_memory.last_hit_position

  IF distance(self, hit_memory.last_hit_position) > 10  action = walk_forward
  ELSE                                                  action = stalk

  acceleration = off, no braking
  voice        = idle
```

**Notes** — four decisions.

*The "route planned since entry" guard is not redundant.* On the first updates the path builder is
still carrying the previous leaf's route, which would read as finished immediately and burn a
destination choice. Comparing the route's plan time against this leaf's entry time is how the leaf
waits for its own route to exist.

*A leg ends at 1.5 units from its end*, not at its end. The creature re-chooses while still moving,
so consecutive legs blend into a continuous advance instead of a stop-start crawl.

*The gait switch at ten units is the behaviour's visible tell.* Outside ten units the creature
walks — audible, upright, covering ground. Inside ten it switches to the stalking gait, which is
slow and quiet. A player who has shot something and then hears it go silent is hearing this line.

*Acceleration is explicitly disabled*, unlike every other advancing leaf in this directory, which
enables it. A stalking animal does not accelerate into a sprint.

The fallback — no cover found, so path straight at the hit point — degrades the behaviour to a
direct approach rather than failing the leaf. That is the right recovery: an animal that cannot
find concealment still comes.

## `check_completion`

**Contract** — finished when a new hit has landed since entry, or when the creature is within three
units of the last hit position.

**Notes** — being hit again is the more important of the two in play. It flips the creature back to
the break-away leaf, so sustained fire keeps a creature oscillating in cover and never lets it
complete an approach. Stopping fire lets it arrive. That trade is the encounter this whole behaviour
exists to create.

Three units is hard-coded and is deliberately inside the ten-unit inner bound of the leg anchor
band: the creature reaches it by drifting in on the final leg, not by aiming at it.
