# src/xrGame/ai/monsters/states/monster_state_squad_rest_inline.h

> Loitering with the pack: flip a coin between standing still for five to ten seconds and ambling
> to a random spot within twenty units of the leader.

**Needs** — [`monster_state_squad_rest.h`](monster_state_squad_rest.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_data.h`](state_data.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../../../restricted_object.h`](../../../restricted_object.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_squad_rest.h`](monster_state_squad_rest.h.md)
**Tier floor** — T3: a coin flip and a three-tier destination search

## Purpose

Selected by the peacetime cascade when the squad leader's standing order is *rest*. Its job is
purely one of appearance: keep the pack visibly clustered and visibly alive. There is no
territorial logic, no cover, no claim — packmates may share ground here, unlike in the solitary
idling behaviour.

The coin flip is the entire high-level design, and it works because the two alternatives have very
different durations. A stand lasts five to ten seconds; an amble lasts as long as the walk takes.
Repeated flipping therefore produces a pack in which some animals are always moving and some are
always still, with no coordination and no schedule.

## State

`Stateless.` The declared reselection-time field is never written or read.

## `reselect_state`

**Contract** — choose stand-still or amble with equal probability, every time a leaf completes.

**Notes** — there is no memory of the previous choice, so a creature can stand twice in a row.
That is the point: an alternating pattern would be visible as a pattern.

## `setup_substates`

**Contract** — fill the parameters of whichever leaf was selected. The amble's destination is
resolved here, at selection time.

```text
stand_still:
  action   = rest
  time_out = random_int(5000, 10000) ms
  voice    = idle, delay = section key "idle_sound_delay"

amble:
  destination = choose_amble_destination()        # below
  gait        = walk forward, accelerating, no braking, calm profile
  completion_dist = 2
  voice       = idle, delay = section key "idle_sound_delay"
```

## `choose_amble_destination` — three tiers

**Contract** — try to find a navigation cell in a ring eight to twenty units around the leader's
cell, giving up after five attempts. Failing that, take a random position within twenty units of
the leader's *position* and, if that position is outside the creature's permitted volumes, replace
it with the nearest position that is inside them.

```text
FUNCTION choose_amble_destination() -> (point, vertex)
  leader = squad.leader
  IF path.random_vertex_in_ring(around = leader.vertex,
                                inner = 8, outer = 20, attempts = 5) SUCCEEDS
    RETURN (navigation.position_of(that vertex), that vertex)

  candidate = random_position_within(leader.position, radius = 20)
  IF NOT restrictions.permits(candidate)
    vertex = restrictions.nearest_permitted(candidate, out point)
    RETURN (point, vertex)
  RETURN (candidate, unknown vertex)
```

**Notes** — the three tiers degrade in a specific way that matters.

*The first tier is a ring with a hole in it.* Eight units is the inner radius, so an amble never
targets the leader's immediate vicinity — packmates spread out rather than converging on the
leader. The ring is sampled by random direction and random radius, bounded to five attempts so the
call is constant-time; this is the same sampling routine the combat fall-back uses.

*The second tier drops the navigation requirement and accepts a raw position*, which the path
builder will resolve to the nearest walkable cell itself. It also drops the inner radius, so a
fallback amble can end up right beside the leader.

*The third tier is the restrictor check*, and it is the only place in this whole peacetime
behaviour that respects the restrictor algebra directly. A candidate position outside the permitted
space is replaced by the nearest position inside it. Everywhere else in peacetime, restrictor
compliance is enforced by the cascade above rather than by individual leaves; here it is enforced
inline because the candidate was invented by random sampling and could be anywhere.

Two units of arrival tolerance ends an amble. The idle window's five-to-ten-second range is what
keeps standing packmates from all starting to move at the same moment.

`idle_sound_delay` is the only authored number; 8, 20, 5, 2, 5000 and 10000 are compiled in.
