# src/xrGame/smart_cover_transition.cpp

> Builds a smart-cover transition edge from its authored table, asks the script layer whether the edge is currently permitted, and picks the animation that lands the creature in the body state the caller wants.

**Needs** — [`smart_cover_transition.hpp`](smart_cover_transition.hpp.md) · [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md) · [`smart_cover_detail.h`](smart_cover_detail.h.md) · [`ai_space.h`](ai_space.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`smart_cover_transition.hpp`](smart_cover_transition.hpp.md)
**Tier floor** — T2: table parsing and a predicate call; no layout or device concern

## Purpose

A smart cover's description holds a graph whose vertices are loopholes and whose edges are
*transitions*. This file owns what sits on an edge. The load-bearing idea is that an edge
is **conditionally traversable**: whether a creature may move from this loophole to that
one is not a property of the geometry, it is a question answered by the authored script
layer at the moment the planner asks. That is why an edge carries a function name and an
argument string rather than a flag.

The second idea is that one edge may carry several animations, distinguished by the
**body state** they leave the creature in (standing, crouching, lying). The planner does
not choose an animation; it chooses a destination and a posture, and this file resolves
that pair to a clip.

## State

```text
RECORD transition_action
  precondition_functor : text                      # script predicate, resolved by name
  precondition_params  : text                      # its single argument
  animations           : list<animation_action>

RECORD authored_edge_table          # the shape this is parsed from
  precondition_functor : text
  precondition_params  : text
  actions : list of
      position      : vector        # offset the creature is placed at for the clip
      animation     : text          # motion name; may be empty (see Notes)
      body_state    : int           # enumerated posture the clip ends in
      movement_type : int           # enumerated gait during the clip
```

## `action` (construction from the authored table)

**Contract** — takes the authored table for one edge; reads the precondition function
name, its parameter string and the list of animation entries; builds one animation record
per entry in authored order. Hard-fails if the value handed in is not a table. Allocates
one record per entry. Does not call the script predicate — construction is parse-only.

```text
FUNCTION build_transition_action(table) -> transition_action
  REQUIRE table IS a table   ELSE FAIL WITH "malformed smart cover transition"
  precondition_functor = table.precondition_functor
  precondition_params  = table.precondition_params
  FOR EACH entry IN table.actions
    APPEND animation_action(entry.position, entry.animation,
                            entry.body_state, entry.movement_type)
```

**Notes** — the entry count is measured before the records are built so the list is sized
once. That is an allocation detail in the original; what survives is that the authored
order is the stored order, because the random pick below is uniform over it and a
different order changes which clip is chosen for a given random draw.

## `applicable`

**Contract** — resolves the edge's precondition by name in the script virtual machine and
calls it with the parameter string, returning its boolean answer. Hard-fails with the
function's name if no such function exists — a missing guard is an authoring error and is
never treated as "permitted". Blocks for the duration of the script call. Called by the
planner during plan search, so it is on a hot path and the script side must be cheap.

```text
FUNCTION applicable() -> bool
  f = script function named precondition_functor
  IF f IS none
    FAIL WITH "failed to get [precondition_functor]"
  RETURN f(precondition_params)
```

**Notes** — resolution happens per call rather than being cached at load. In a rebuild it
is legitimate to resolve once and store a handle, provided a script reload invalidates it:
the shipped scripts may replace a global function between level loads, and the engine's
behaviour of looking it up fresh is what makes that work.

## `animation` (by body state)

**Contract** — returns the first animation of this edge whose end posture equals the
requested one. If none matches, logs a diagnostic in non-shipping builds and **falls back
to a random animation of the edge** rather than failing. Never returns nothing: the edge
is guaranteed non-empty by construction.

```text
FUNCTION animation(target_body_state) -> animation_action
  FOR EACH a IN animations
    IF a.body_state IS target_body_state
      RETURN a
  RETURN animation()        # degrade rather than stall the creature
```

**Invariants** — `animations` is non-empty, which is what makes the fallback safe.

**Notes** — the fallback is a deliberate robustness choice about *shipped data the engine
does not own*: an authored cover that is missing, say, the crouching variant of a
transition would otherwise deadlock a creature mid-cover. Playing the wrong posture's clip
is visibly imperfect and recoverable; stalling is not. A rebuild that hard-fails here will
fail on shipped content.

## `animation` (random)

**Contract** — returns a uniformly random animation of this edge. Used both directly, when
the caller has no posture preference, and as the fallback above.

**Notes** — the draw comes from the global random stream, not a per-cover one, so the
choice is not reproducible from the cover's own state. Conformance criterion 8 asks only
that physics be deterministic, not that animation selection be, so this is within budget —
but a rebuild aiming for replayable AI should route this through the same seeded stream as
the rest of the simulation.
