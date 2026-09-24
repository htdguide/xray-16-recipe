# src/xrGame/ai/monsters/monster_home.cpp

> The place a creature belongs to: an authored patrol path or a single vertex surrounded by three nested radii, with six different ways of picking a destination inside it depending on whether the creature is settling, wandering, or backing away from something.

**Needs** — [`monster_home.h`](monster_home.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`monster_cover_manager.h`](monster_cover_manager.h.md) · [`cover_point.h`](../../cover_point.h.md) · [`restricted_object.h`](../../restricted_object.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../../../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path_storage.h`](../../../xrAICore/Navigation/PatrolPath/patrol_path_storage.h.md)
**Used by** — [`monster_home.h`](monster_home.h.md)
**Tier floor** — T3: radii and random sampling over the navigation graph

## Purpose

Most creatures in the shipped levels are not free-roaming: each is authored with a *home*, and
its idle behaviour, its willingness to chase, and where it retreats to are all expressed
against that home. This file is the home and the destination picker.

A home is one of two shapes, and every query branches on which:

- **A patrol path** — an authored waypoint graph. Containment means "within the radius of *any*
  waypoint", so a path home is a corridor or a blob, not a disc. Destination picks start from a
  uniformly random waypoint, so a long path spreads the creature along it.
- **A single navigation vertex** — a disc. Simpler in every query.

Around either shape sit **three radii**, inner, middle and outer, and they are what makes the
home more than a leash. Roughly: the inner radius is the creature's den, the ring between inner
and middle is where it wanders while idle, and the outer radius is the boundary past which it
will not chase. Which radius a behaviour consults is the behaviour's whole character.

A home also carries an **aggressive** flag, which this file only stores; the brains read it to
decide whether crossing into the home is provocation.

## State

```text
RECORD MonsterHome
  creature        : BaseMonster
  path            : optional<PatrolPath>   # set means the home is a path
  vertex          : int                    # otherwise the home is this single vertex; INVALID if neither
  inner_radius    : real                   # default 20
  middle_radius   : real                   # default 30; invariant: inner <= middle <= outer
  outer_radius    : real                   # default 40
  wander_min      : int                    # default 7  — the near edge of one idle step
  wander_max      : int                    # default 10 — the far edge; invariant: min < max
  aggressive      : bool
```

Invariants, all enforced at load and at setup rather than at use: the outer radius must exceed
the inner one or the creature fails to load; a middle radius outside `[inner, outer]` is
silently replaced by their midpoint, as is an absent one; a wander band with `min >= max` is
silently replaced by the default pair. The silent repairs mean bad authoring produces a
plausible home rather than a broken creature.

`has_home` requires *both* a path and a valid vertex, which no shipped configuration produces —
see the Notes on `load`.

## `load`

**Contract** — reads the home from a named section of the creature's spawn configuration. Sets
the defaults first, so a creature whose spawn record has no such section still has a home shape
(no path, no vertex) that every containment query answers *yes* to. Reads the patrol path by
name — required if the section exists — then four optional numbers. Not aggressive.

```text
FUNCTION load(section_name)
  path = none; vertex = INVALID
  inner, middle, outer   = 20, 30, 40
  wander_min, wander_max = 7, 10

  IF the spawn configuration has this section
    path = patrol_path_registry.lookup(section.path)      # required
    inner = section.radius_min IF present
    outer = section.radius_max IF present
    FAIL WITH "wrong home radii" IF outer <= inner

    middle = section.radius_middle IF present, else the midpoint of inner and outer
    IF middle outside [inner, outer] THEN middle = the midpoint

    wander_min = section.min_move_dist IF present
    wander_max = section.max_move_dist IF present
    IF wander_min >= wander_max THEN restore 7 and 10

  aggressive = false
```

**Notes** — the five authored keys are `path`, `radius_min`, `radius_max`, `radius_middle`,
`min_move_dist`, `max_move_dist`, all in the creature's *spawn* configuration rather than its
class section, because a home is per-placement, not per-species.

A creature loaded this way has a path and an invalid vertex, so `has_home` — which demands both
— answers no even though every other query behaves as though there is a home. Every caller that
matters asks the containment and placement queries instead, so the inconsistency is latent; it
is not resolvable from the source whether `has_home` meant "either" and was written as "both".

A debug-only check asserts that the named patrol path actually lies on the level the creature
is on, with a message naming both levels. It catches a real authoring failure — a home pointing
at another level silently paths creatures nowhere — and a rebuild should keep the check, in
release too.

## `setup`

**Contract** — two script entry points establishing a home around a named patrol path or around
a single navigation vertex, with explicit radii and the aggressive flag. The middle radius is
repaired to the midpoint if it falls outside the other two, exactly as at load. The path form
does *not* touch the stored vertex and the vertex form clears the path, so the two forms are
not symmetric: setting a path home after a vertex home leaves the old vertex in place, where
`has_home` will then see both and answer yes.

## `clear_home`

**Contract** — drops both shapes and the aggressive flag. A creature with no home is
unconstrained: every containment query answers yes.

## Containment

**Contract** — `contains(position, radius)` is the primitive; the rest name a radius.

```text
FUNCTION contains(position, radius) -> bool
  IF no path
    IF vertex is INVALID THEN RETURN true          # no home means everywhere is home
    RETURN distance(position, vertex_position(vertex)) < radius

  FOR EACH waypoint IN path
    skip waypoints whose navigation vertex is invalid
    IF distance(position, vertex_position(waypoint.vertex)) < radius THEN RETURN true
  RETURN false
```

`contains()` with no argument tests the creature's own position against the **outer** radius;
`within_inner` and `within_middle` name the other two. So "at home" without qualification means
"inside the outer boundary".

**Notes** — a path home is a union of discs, one per waypoint, so its shape follows the path.
A path whose waypoints are further apart than twice the radius has gaps in it, and a creature
standing in a gap is not at home. Nothing detects that; it is an authoring constraint.

## Destination queries

Six of them, each answering a navigation vertex. They share a skeleton — pick a seed (a random
waypoint, or the single vertex, or the creature's own position as a fallback), ask the
creature's path builder for a *reachable* vertex within a radius band of the seed, and fall back
when it finds none — and differ in which band and which fallback.

The shared fallback is worth stating once, because five of the six use it: if no reachable
vertex was found, return the seed if the creature can reach the seed, otherwise return where
the creature is already standing. A creature is always given *some* destination, never a
failure; standing still is the last resort.

The sample count handed to every radius query is five — five attempts to find a reachable
vertex in the band before giving up. Nothing derives it.

### `place_in_inner`

**Contract** — a reachable vertex within the inner radius of the seed. The band is `[1, inner]`,
so it is a disc with a one-unit hole rather than a ring. Used to send a creature to its den.

### `place_in_middle` — the wander step

**Contract** — the idle-movement pick, and the only one with two distinct modes.

```text
FUNCTION place_in_middle() -> vertex
  step = wander_min + random below (wander_max - wander_min)

  IF the creature is OUTSIDE the middle radius, OR INSIDE the inner one
    # it is in the wrong place: send it into the wander ring
    seed   = a random waypoint, or the home vertex, or where it stands
    result = reachable vertex in the band [inner, middle] around seed
  ELSE
    # it is already in the ring: take one step forward-ish
    REPEAT up to 10 times
      turn = random within a quarter turn either side of the creature's facing
             (after 5 failures, widen to between a quarter and a third of a turn, either side)
      candidate = creature.position + step units along that turn
      UNTIL candidate lies on the navigation graph
    IF candidate is on the graph
      result = the vertex at candidate
    ELSE
      result = reachable vertex in the band [step-1, step] around where it stands

  apply the shared fallback if result is still nothing
  IF result is invalid OR outside the MIDDLE radius THEN RETURN place_in_inner()
  RETURN result
```

**Notes** — the two modes are the file's most consequential decision. A creature that is
*already* in its wander ring does not get a random point in the ring; it takes one step of
authored length, biased to *continue roughly forward*, which is what makes idle creatures
appear to be going somewhere rather than teleporting between random spots. The bias is a random
turn within a quarter turn either side of current facing; only after five failures does it
widen and force a turn of at least a quarter, which is the escape from a dead end.

The final guard sends any result outside the middle radius back to the inner query, so a wander
step can never leave the ring — the wander band is a hard boundary even though the step
direction is unconstrained.

### `place_towards`

**Contract** — a reachable vertex in the outer part of the home, in a supplied direction from
the home's anchor. This is the "back away that way but stay home" pick.

```text
FUNCTION place_towards(direction) -> vertex
  anchor = anchor_position()
  reach  = midpoint of (middle, outer), pulled in by a tenth of their gap

  REPEAT up to 10 times
    turn = the supplied direction, jittered by up to a fifth of a turn either side
           (after 5 failures, forced to between a fifth and a quarter of a turn, either side)
    candidate = anchor + reach units along that turn
    UNTIL candidate lies on the navigation graph

  IF candidate is on the graph
    result = reachable vertex within (outer - middle)/2 of it

  IF nothing found
    IF the candidate vertex is reachable THEN result = it
    ELSE retry the whole thing at the INNER reach — the midpoint of (inner, middle),
         pulled in by a tenth, with a wider jitter of up to a third of a turn
    IF that also fails THEN result = place_in_inner()

  IF still nothing THEN result = place_in_outer()
  RETURN result
```

**Notes** — two rings are tried, outer then middle, each with its own jitter width, and only
then does it fall back to the den. The "pulled in by a tenth of the gap" on each reach keeps
the target just inside the ring rather than exactly on its boundary, where a rounding error
would put it outside the home.

The whole routine is one decision — *stay inside the home while moving in the requested
direction* — expressed twice at two radii. A rebuild should factor the ring attempt into one
named step taken twice.

### `place_in_outer`

**Contract** — a reachable vertex in the band `[inner, outer]` around the seed, with the shared
fallback. The widest wander.

### `place`

**Contract** — the general-purpose pick, in the band from the inner radius to the midpoint of
inner and outer. Its fallback chain is different from the shared one and is the most forgiving
in the file: for a vertex home, if the band fails it tries a band of `[5, 15]` around the
creature, then `[2, 3]` with ten samples instead of five, and only then gives up and stands
still. For a path home it uses the shared fallback.

**Notes** — the escalating fallback exists because this is the query used when a creature must
move *now* and the surrounding navigation may be crowded. The three bands read as
"anywhere in the home", "anywhere nearby", "anywhere at all within two paces".

### `place_in_cover`

**Contract** — the best cover point within the band from the inner radius to the midpoint of
inner and outer, around a random waypoint or the home vertex, hiding *from that same point*.
Unlike every other query here it can genuinely fail, answering nothing rather than falling back.

**Notes** — passing the seed as both the search origin and the thing to hide from means "find
cover near home from the direction of home", which is not obviously what a hiding creature
wants; the effect is to prefer points near home that are concealed from home's own centre. No
rationale is recoverable.

## `anchor_position`

**Contract** — one point representing the home: the home vertex's position, or the position of
the **first** waypoint of the path, or — if neither exists — wherever the creature is standing.

**Notes** — for a path home this is the first waypoint, not a centroid, so `place_towards`
measures its reach from one end of the path rather than its middle. On a long path that skews
every directional pick towards that end.

## `set_wander_step`

**Contract** — sets the wander band from script. Silently ignores a band with `max <= min`,
leaving the previous pair in place.
