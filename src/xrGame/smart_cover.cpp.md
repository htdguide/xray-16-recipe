# src/xrGame/smart_cover.cpp

> Places one authored cover on a level: binds each enabled loophole to a navigation vertex, and answers the question the AI actually asks — which loophole should I use against a target standing there.

**Needs** — [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_storage.h`](smart_cover_storage.h.md) · [`smart_cover_object.h`](smart_cover_object.h.md) · [`smart_cover_action.h`](smart_cover_action.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: navigation-vertex lookups at placement, a short scored scan per query

## Purpose

A description says what a cover *is*; this file says where it is and which of its
loopholes this particular placement offers. Two jobs live here.

The first is placement: for each enabled loophole, find the navigation vertex a creature
would stand on to use it, and do the same for every action that moves the creature
somewhere else. This is done **once, at construction**, because the level graph lookup is
a spatial query and the planner asks for these vertices constantly.

The second is selection: given a world position — almost always an enemy's — score every
enabled loophole and return the best. There are two scoring rules and they are not
variations of one another; see below.

## State

Declared in [`smart_cover.h`](smart_cover.h.md).

## Construction

**Contract** — takes the placed entity, the shared description, the two fire flags, and an
authored table of per-loophole availability. Builds the enabled loophole list, then the
parallel vertex list. Fails if any loophole's navigation vertex is invalid on the loaded
level, naming the cover and the loophole. Registers itself as a cover point at the placed
entity's own position and navigation vertex, flagged as a smart cover.

```text
FUNCTION place_cover(object, description, is_combat_cover, can_fire, availability)
  base cover point = (object.position, object.level_vertex)
  mark as a smart cover
  IF availability IS given
    FOR EACH loophole IN description.loopholes            # description order is preserved
      IF availability names this loophole
        IF availability says false THEN SKIP
        ENABLE it
  ELSE
    enable every loophole of the description
  FOR EACH enabled loophole
    bind_vertices(loophole)
```

**Invariants** —

- The enabled list keeps the **description's order**, not the availability table's. The
  availability table is a map from loophole name to a flag; iterating it would give an
  order that varies between runs, and `best_loophole` breaks ties by first-encountered.
- A loophole the availability table does not mention at all is **enabled**. Only an
  explicit false disables it. That is what lets an author disable one loophole of a
  ten-loophole cover with a one-entry table.
- Missing availability entirely enables everything.

**Notes** — the availability table is checked by scanning it per loophole rather than
looking up by name, which is quadratic in the loophole count. Loophole counts are single
digits, so it does not matter; a rebuild should still use a lookup.

## `vertex` (binding a loophole to the navigation graph)

**Contract** — computes the navigation vertex for a loophole and for each of its actions
that moves the creature. Both are found by taking the authored local position, transforming
it to world space, **raising it by two metres**, and asking the level graph which vertex
contains it. Fails if any result is not a valid vertex.

```text
FUNCTION bind_vertices(loophole) -> loophole_data
  p = world position of loophole.fov_position, raised 2 m
  level_vertex_id = the level graph vertex containing p    # must be valid
  FOR EACH (name, action) IN loophole.actions
    IF action does not move the creature THEN SKIP
    q = world position of action.target_position, raised 2 m
    action_vertices[name] = the level graph vertex containing q   # must be valid
```

**Invariants** — the two-metre lift is the load-bearing detail. The authored position is
where a creature's *eye* is, which for a crouching or leaning pose can be below, inside or
behind the geometry the navigation mesh was built for; the level graph's containment query
searches downward, so lifting the probe above the cover and letting it fall finds the floor
vertex rather than failing or finding a vertex on the wrong side of a wall. A rebuild whose
navigation query is not a downward search will need its own answer to the same problem, but
it must still be *one* answer applied identically to loopholes and to action targets, or a
moving action will disagree with the loophole it belongs to.

**Notes** — the constructor computes each loophole's vertex twice, once inline and once
inside this routine, discarding the first. That is redundant work at level load, not a
decision.

## `best_loophole`

**Contract** — scores every enabled loophole against a world position and returns the best,
writing the winning score out. Yields nothing when no loophole qualifies — a cover can be
placed and have no usable answer against a given target, and the caller must handle that.
Selects between the two scoring rules by the caller's flag.

```text
FUNCTION best_loophole(position, use_default_behaviour, is_entered) -> (loophole, value)
  value = +infinity
  result = none
  FOR EACH loophole IN enabled loopholes
    IF use_default_behaviour
      score_for_default_usage(position, loophole, result, value)
    ELSE
      score_for_combat(position, loophole, result, value, is_entered)
  RETURN (result, value)
```

**Invariants** — both rules improve strictly, so the first loophole of the description's
order wins a tie. Order is therefore authoring-significant.

## `evaluate_loophole` — the combat rule

**Contract** — admits a loophole only if it is usable, the target is within range, the
target is **not too close**, the target is at least a metre away, and the target lies
inside the loophole's arc. Scores by angular offset normalized against the arc width, so
the score is comparable between loopholes with different arcs.

```text
FUNCTION score_for_combat(position, loophole, best, value, is_entered)
  IF NOT loophole.usable                                  RETURN
  eye = world fov_position of loophole
  d   = distance from eye to position
  IF d > loophole.range                                   RETURN
  min_distance = object.enter_min_enemy_distance IF is_entered
                 ELSE object.exit_min_enemy_distance
  IF d <= min_distance                                    RETURN
  direction = position - eye
  IF |direction| < 1 m                                    RETURN
  normalize direction
  alpha = |angle between loophole's world fov_direction and direction|
  IF alpha >= loophole.fov / 2                            RETURN
  IF alpha >= value                                       RETURN
  value = 2 * alpha / loophole.fov
  best  = loophole
```

**Invariants** —

- The minimum-enemy-distance threshold **differs depending on whether the creature is
  already in the cover**. A creature inside must tolerate a closer enemy before abandoning
  the cover than it would have accepted to enter it in the first place; without the
  asymmetry a creature would enter and immediately leave as the enemy closed, oscillating
  in the doorway. Both thresholds are per-placement settings on the cover object
  ([`smart_cover_object.cpp`](smart_cover_object.cpp.md)).
- The one-metre floor on the direction length is separate from the minimum-enemy-distance
  test and guards the *angle*: at sub-metre separation the direction is dominated by noise
  and the arc test becomes meaningless.
- The comparison that selects the winner uses the **raw angle**, while the value written
  out is the **normalized** one. They are not the same ordering, so a loophole with a
  narrow arc can lose to one with a wide arc that is angularly worse relative to its own
  arc. This is an inconsistency in the original, and it is load-bearing in the sense that
  reproducing the behaviour requires reproducing it: fixing it changes which loophole
  creatures pick in shipped covers.

## `evaluate_loophole_for_default_usage` — the non-combat rule

**Contract** — admits any usable loophole and scores purely by angular offset from the
loophole's facing to the position. No range test, no arc test, no minimum distance. The
best loophole is simply the one pointing most nearly at the position.

**Invariants** — the direction is normalized with a degenerate-safe normalization rather
than rejected when short, because with no distance floor the target may legitimately be at
the loophole's own eye point. The consequence is that the angle can be arbitrary in that
case; it is accepted because a non-combat cover's choice of loophole has no consequence
worth guarding.

**Notes** — the split into two rules is the real content of this file. Combat selection is
"can I shoot the enemy from here, and how squarely"; default selection is "which way is
the thing I care about, roughly". A rebuild that unifies them will make creatures refuse to
use non-combat covers whenever the thing they are looking at is out of range.

## `level_vertex_id` / `action_level_vertex_id`

**Contract** — return the precomputed navigation vertex for a loophole, and for a named
moving action of a loophole. Both fail rather than yield nothing: the caller has already
selected the loophole and the action, so a miss means the planner and the placement
disagree.

**Notes** — both search the parallel vertex list linearly, matching loopholes by identity
and action names by interned-string identity rather than by content.

## Arc and range predicates

**Contract** — four tests the planner's evaluators ask about a world position:

- **in field of view** — inside the loophole's wide arc, with the same one-metre floor and
  half-angle comparison as the combat rule.
- **in danger field of view** — the same test against the narrower danger arc and its own
  direction. This is what distinguishes "I can see him from here" from "he is pointing at
  me".
- **in range** — within the loophole's reach.
- **in minimum acceptable range** — *at least* the caller's distance away; the
  complementary half of the range test.

**Invariants** — all four transform the loophole's local geometry to world space on each
call ([`smart_cover_inline.h`](smart_cover_inline.h.md)), so they reflect a cover object
that has moved.

## Connectivity checking (development builds)

**Contract** — after placement, in development builds only, the cover asserts that every
ordered pair of its enabled loopholes is connected by a path through the description's
transition graph, and that every loophole is reachable from the reserved enter vertex and
can reach the reserved exit vertex. The path search uses the general graph engine with all
limits removed.

**Invariants** — this is the guarantee the animation planner relies on and never rechecks:
a creature that enters a cover can reach any loophole of it and can always leave. Because
the check runs over the *enabled* subset, an availability table that disables a loophole
acting as the only bridge between two others is caught here and nowhere else.

**Notes** — a rebuild should keep this check and, unlike the original, keep it in shipping
builds as a one-time cost at level load. A cover that fails it does not crash; it strands a
creature, which is far harder to diagnose later than a message at load time.
