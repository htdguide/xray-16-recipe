# src/xrGame/ai/monsters/monster_cover_manager.cpp

> Turns the level's precomputed per-vertex cover values into two creature-level answers: the best place to hide from a threat at a chosen distance, and the most open direction to face.

**Needs** — [`monster_cover_manager.h`](monster_cover_manager.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`cover_evaluators.h`](../../cover_evaluators.h.md) · [`cover_point.h`](../../cover_point.h.md) · [`cover_manager.h`](../../cover_manager.h.md) · [`ai_monster_squad.h`](ai_monster_squad.h.md) · [`ai_monster_squad_manager.h`](ai_monster_squad_manager.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../xrAICore/Navigation/level_graph.h.md) · [Seam: Static collision database](../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`monster_cover_manager.h`](monster_cover_manager.h.md)
**Tier floor** — T2: a bounded search over precomputed level data plus a handful of rays

## Purpose

The level ships a per-navigation-vertex *cover* measure: for each vertex and each compass
direction, how exposed that vertex is from that direction, in a high variant and a low one.
The engine's cover manager can enumerate candidate cover points near a position. Neither knows
what a creature wants.

This file supplies the creature's wants as a *scoring policy* handed to that enumeration —
where to hide *from*, at what distance band, and which points are already claimed by a
pack-mate — and separately answers a question the cover data can also serve backwards: which
direction is the most open, used for facing when a creature wants to watch approaches rather
than hide from one.

## State

```text
RECORD CoverManager
  creature  : BaseMonster
  evaluator : CoverEvaluator        # long-lived; rebuilt only at load

RECORD CoverEvaluator                # the scoring policy handed to the level's cover search
  creature          : BaseMonster
  threat_position   : vector         # what we are hiding from
  min_distance      : real           # the near edge of the wanted band
  max_distance      : real           # the far edge
  deviation         : real           # carried but unread — see Notes
  start_distance    : real           # distance from the search origin to the threat, at setup
  best_value        : real           # lowest score seen; lower is better
  selected          : optional<CoverPoint>
```

Invariant: the evaluator is bound at `load` to the creature's *movement restrictions*, so every
point it is offered is already one the creature is permitted to occupy. A rebuild that skips
that binding gets creatures hiding outside their restrictor.

## `find_cover`

**Contract** — asks the level's cover manager for the best point within a fixed radius of a
search origin, scored by this creature's policy against a threat position. Answers nothing when
no candidate passes. Two forms: the short one searches around the creature itself, the long one
around a supplied origin, which is how a creature picks cover near a place it has not reached
yet.

```text
FUNCTION find_cover(origin, threat, min_distance, max_distance, deviation) -> optional<CoverPoint>
  evaluator.setup(creature, threat, min_distance, max_distance, deviation)
  RETURN level_cover_search(origin, SEARCH_RADIUS, evaluator)
```

**Notes** — the search radius is thirty world units in both forms, hard-coded, and is the
bound on how far a creature will look for cover. It is not the same thing as the distance band:
the band says how far the *point* should be from the threat, the radius says how far it may be
from the *creature*.

The `deviation` parameter is plumbed all the way through and stored, but nothing reads it. It
participates only in the evaluator's staleness check — a query with a different deviation
counts as a changed query and re-runs — so it behaves as an opaque cache key. Whether it once
meant an angular tolerance is not recoverable.

## `CoverEvaluator.score` — what makes one cover point better than another

**Contract** — called by the level's cover search once per candidate point, with a weight the
enumeration attaches. Rejects outright, or records the point if it scores lower than the best
seen. Lower is better.

```text
FUNCTION score(point, weight)
  IF creature.squad().cover_is_claimed(point.vertex) THEN RETURN   # a pack-mate is going there
  IF weight is zero THEN RETURN

  d = straight_line(threat_position, point.position)

  # reject a point that is both outside the wanted band AND takes us the wrong way:
  IF d <= min_distance AND start_distance > d THEN RETURN   # too close, and closer than we were
  IF d >= max_distance AND start_distance < d THEN RETURN   # too far,  and further than we were

  heading = compass heading from point towards the threat
  value   = min( high_cover(point.vertex, heading), low_cover(point.vertex, heading) )
  IF the vertex has a walkable neighbour in that direction
    value = value + 10                       # penalty: an open approach spoils the cover

  value = value / weight

  IF value < best_value
    best_value = value; selected = point
```

**Invariants** — a point is only rejected for being outside the band when accepting it would
move the creature *further from where it wants to be*. A point that is too close but still an
improvement on where the search started is kept. That asymmetry is what lets a cornered
creature take imperfect cover rather than find nothing.

**Notes** — the cover measure used is the *worse* of the high and low variants, so a point only
scores well when a creature is concealed both standing and crouched. Taking the minimum is a
deliberate conservatism.

The penalty of ten for a walkable neighbour towards the threat is the one unexplained constant.
Cover values are small numbers, so ten is effectively a disqualification rather than a weight:
a vertex the threat can simply walk onto is never chosen if any alternative exists. The
magnitude is arbitrary; the intent is not.

Dividing by the weight means the enumeration's own per-point preference multiplies the score
inversely — a point the level considers twice as good needs only half the concealment.

Smart covers (the authored, animated hiding places stalkers use) are offered to this evaluator
and ignored: creatures do not use them.

## `least_cover_direction`

**Contract** — writes a heading pointing at the most *open* nearby direction, used when a
creature wants to face where something could come from rather than hide from something known.
Casts up to a bounded number of short rays; allocates nothing.

```text
FUNCTION least_cover_direction() -> direction
  # start from the direction in which this vertex's high cover is greatest, sampled every 10 degrees
  angle = vertex_extreme_cover_angle(creature.vertex, step = 10 degrees, pick = greatest)

  left  = angle - QUARTER_TURN            # the widest arc we will consider
  right = angle + QUARTER_TURN

  # walk left in 10-degree steps until a short ray hits static geometry; that is the left wall
  FOR ang FROM angle SWEEPING LEFT WHILE within a quarter turn of angle
    IF a ray from the creature's centre along ang hits static geometry within TRACE_DISTANCE
      left = ang; BREAK

  # same to the right
  FOR ang FROM angle SWEEPING RIGHT WHILE within a quarter turn of angle
    IF a ray from the creature's centre along ang hits static geometry within TRACE_DISTANCE
      right = ang; BREAK

  RETURN heading at the midpoint of the arc between left and right
```

**Notes** — the seed is the direction of *greatest* cover, which reads backwards for a routine
named "least cover", and then the rays narrow the arc to what is actually unobstructed and the
answer is that arc's middle. The resulting behaviour is "face down the open lane you are
standing in", which is what the callers want; the seed is best read as "start from the
direction the walls are behind you".

Three constants: the arc is a quarter turn each way, the angular step is ten degrees, and the
rays run three world units. The step and the arc together bound the work at nine rays per side.
Nothing derives any of the three.

If neither sweep hits anything, the arc stays at the full half-turn and the answer is the seed
direction itself — the midpoint of a symmetric arc.
