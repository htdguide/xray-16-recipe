# src/xrGame/moving_objects_static.cpp

> Finds the level furniture a creature's next second of walking would run into, and records it as that creature's static obstacle set.

**Needs** — [`moving_objects.h`](moving_objects.h.md) · [`moving_object.h`](moving_object.h.md) · [`moving_objects_impl.h`](moving_objects_impl.h.md) · [`obstacles_query.h`](obstacles_query.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: proximity queries and path sampling

## Purpose

The half of obstacle avoidance that deals with things that do not move but are not level
geometry either: crates, corpses, parked vehicles, anything the level's static collision
database does not know about because it was spawned. The engine calls these *static* even
though they are dynamic objects, because from the pathfinder's point of view they are
stationary furniture.

Its output is not a decision but a *set*: the objects in the way. Something else — the
pathfinder's cost model — reads that set and routes around them.

## State

`Stateless` — it fills the system's scratch list and the creature's static obstacle set.

## `query_action_static`

**Contract** — the entry point. Given a creature (and optionally an explicit start and
destination; otherwise its current position and its position predicted one second ahead),
fills that creature's static obstacle set with every nearby object that would obstruct it.
Leaves the set untouched when nothing is in the way. Two proximity queries in the worst
case, one in the common case. Allocates only into reused scratch.

```text
FUNCTION query_action_static(creature, start, destination)
  midpoint := average(start, destination)
  radius   := distance(midpoint, destination)           # the segment's half-length

  fill_nearest(midpoint, radius, creature)              # cheap first pass
  IF nothing nearby THEN RETURN                         # the overwhelmingly common case

  IF NOT collided_static(creature, destination) THEN RETURN

  # something really is in the way: re-query with a margin, because an object
  # that merely grazes the segment must also be avoided
  fill_nearest(midpoint, radius + additional_radius, creature)
  add every object in the scratch list to the creature's static obstacle set

FUNCTION query_action_static(creature)
  query_action_static(creature, creature position, creature predicted one horizon ahead)
```

**Invariants** — the query is centred on the *midpoint* of the movement segment with a
radius of half its length, which is the smallest sphere containing the segment. This is
what keeps the common case cheap: most creatures are walking through empty space and the
first query returns nothing.

The second query is widened by a fixed two metres rather than being the same query reused.
That widening is not paranoia: the first query's sphere touches the segment's endpoints
exactly, so an object just outside it can still obstruct a creature of non-zero radius
walking near the end of the segment.

**Notes** — once an obstruction is confirmed, *every* object from the widened query is added
to the set, not only the ones that actually collide. A precise version exists in the source
— it samples the path and adds only the objects hit at each sample — and is commented out
in favour of the blunt one. The blunt version makes the pathfinder route around a slightly
larger region than strictly necessary, which is cheap and conservative; the precise version
costs a per-sample loop over the candidate list. A rebuild may choose either.

## `collided_static`

**Contract** — answers whether a creature following its predicted path would come within
its own radius (plus half a navigation cell) of any object in the scratch list, sampling
the path at fixed distance intervals. Reads only.

```text
FUNCTION collided_static(creature, destination) -> bool
  radius   := creature radius + half a navigation cell
  velocity := distance(creature position, destination) / horizon
  samples  := round(velocity * horizon / step_to_check)

  FOR i IN 0 .. samples-1
    point := creature position        IF i is the first sample
           | destination              IF i is the last sample
           | creature predicted at (i * step_to_check) seconds otherwise
    IF any scratch object's footprint reaches (point, radius) THEN RETURN true
  RETURN false
```

**Invariants** — the creature's radius is inflated by half a navigation cell. A creature
whose centre is on a walkable vertex may legitimately stand anywhere within that cell, so
avoiding only its nominal radius would let it clip furniture at the cell's edge.

**Notes** — the sample count is computed as *velocity × horizon ÷ step*, but velocity was
itself computed as *distance ÷ horizon*, so the horizon cancels and the count is simply the
segment's length divided by the step. The two-step derivation is left as written because
the intermediate velocity is the quantity the constants are tuned against; a rebuild may
simplify it to a length division without changing behaviour.

The intermediate samples are taken as *predicted* positions at *i × step* **seconds**,
while the loop's step is a **distance**. The two units are being conflated: with a step of
half a metre, sample *i* asks where the creature will be in *i* half-seconds, not in *i*
half-metres. For a creature moving at one metre per second they coincide, which is roughly
the shipped walking speed, so the error is small in practice and grows with speed. A
rebuild should convert the step to a time by dividing by the velocity.

## `fill_nearest_list`

**Contract** — asks the world's spatial object registry for everything within a radius of a
point, excluding the creature itself, then removes from the result every object the
creature ignores and every object not marked as an AI obstacle in its configuration.
Writes into the shared scratch list.

**Invariants** — the AI-obstacle flag is a per-object property from configuration, not a
physical one. Most spawned objects are not obstacles: dropped items, decorations and
projectiles are walked through. A rebuild that treats every physical object as an obstacle
will make creatures refuse to cross rooms.

## `fill_static`

**Contract** — two forms. The unfiltered one adds every object in the scratch list to a
query set. The filtered one adds only those whose footprint reaches a given (position,
radius) circle.

**Notes** — the filtered form and the sampled `fill_all_static` that uses it are the precise
path the entry point does not take. They are kept because they are the same loop as the
collision test with "record" substituted for "return true", and re-enabling precision is
a one-line change at the call site.
