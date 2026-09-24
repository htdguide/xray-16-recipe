# src/xrGame/alife_monster_abstract.cpp

> The offline creature: its registration and promotion hooks, the rule that decides when a corpse may finally be forgotten, and the population model that lets a pack breed while nobody is watching.

**Needs** — [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_group_registry.h`](alife_group_registry.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`alife_monster_brain.h`](../xrServerEntities/alife_monster_brain.h.md) · [`alife_monster_movement_manager.h`](alife_monster_movement_manager.h.md) · [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`ef_storage.h`](ef_storage.h.md) · [`ef_pattern.h`](ef_pattern.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: lifecycle hooks and two probabilistic rules.

## Purpose

The base behaviour of every non-player creature's server record. Most of it is forwarding to
the creature's brain, but three decisions live here and only here: what makes a creature
*active* at all, when a dead creature's record may be discarded, and how a group's
population grows over time.

## `update`

**Contract** — Skips inactive records; otherwise runs the brain for one coarse step. That is
the entire live behaviour.

**Notes** — The original off-screen *wandering* is commented out immediately below it, and is
worth recording because it is the mechanism the word *alife* names. Its shape:

```text
# Disabled: a creature travelling the cross-level graph while nobody watches.
WHILE the creature is still active and still has ground to cover
  check_for_population_changes()
  IF it is moving and has a destination vertex
    # Advance along the current edge by elapsed game time times its speed.
    distance_covered += (now - last_stamp) / time_scale * speed
    IF it has reached the destination
      carry the overshoot forward as a corrected timestamp, so no distance is lost
      previous vertex = current vertex
      move it to the destination vertex in the graph registry
      IF it is a group, mark it as needing fresh spawn positions next expansion
  IF it is moving and has arrived
    # Choose the next edge: only vertices whose terrain matches the creature's
    # allowed terrain masks are candidates, and the vertex it came from is excluded
    # unless nothing else qualifies.
    count the qualifying outgoing edges that are not the way it came
    IF none qualify
      take the first qualifying edge including the way it came
    ELSE
      take a uniformly random one of the qualifying edges
    IF still nothing qualifies
      stop moving
    ELSE
      resume at its configured travel speed
  IF it changed vertex, it would have been offered for interaction here
```

The three recoverable decisions in that disabled loop: travel time is *game* time divided by
the world's time scale, so the off-screen world moves at the clock's pace and not the
frame's; overshoot past a vertex is carried forward rather than discarded, so a creature
crossing several short edges in one update does not lose distance; and backtracking is
forbidden unless it is the only option, which is what makes a wandering creature go
*somewhere* instead of oscillating on one edge.

## `bfActive`

**Contract** — A creature counts for the simulation when it is interactive and either — for a
group record — has at least one member, or — for an individual — has health above zero.

**Invariants** — This single predicate gates the update, the interaction pass and several
registry decisions. The group and individual cases are genuinely different questions, which
is why the test is here rather than on health alone.

## `on_register` / `on_unregister` / `add_online` / `add_offline` / `on_location_change`

**Contract** — Each runs the base's own handling and then notifies the brain, in that order —
the brain reads registry state that must already be correct.

`on_unregister` does three extra things, in order: clears every reputation relation naming
this creature, notifies the brain, and removes it from its group if it has one.

**Invariants** — Relations are cleared *before* the group membership is dropped. A relation
registry entry naming a dead creature is a dangling reference in exactly the sense §6
forbids, and the group's own bookkeeping may consult relations while unregistering a member.

## `tpfGetBestWeapon`

**Contract** — A creature *is* its weapon: no weapon object is returned, and the outputs are
the creature's own configured hit power and hit type.

## `tpfGetBestDetector`

**Contract** — An individual creature is its own detector; a group answers with its first
member, or with nothing when empty. Creatures have no instruments, so "what would it search
with" collapses to "itself".

## `tfGetActionType`

**Contract** — As shipped, a creature meeting anything ignores it. The real rule is disabled:
friends ignore each other, a mutual sighting or a favourable odds estimate means attack, and
anything else is ignored; meeting a smart terrain defers the decision to the terrain.

## `vfCheckForPopulationChanges` — breeding

**Contract** — Grows a group's population over game time. Applies only to group records that
are active and offline; an expanded group does not breed. Reads four tuned evaluation
functions.

```text
FUNCTION check_for_population_changes()
  IF not a group, or not active, or currently online THEN RETURN
  IF the game clock has not reached the group's next birth time THEN RETURN

  # Schedule the next opportunity: the birth-speed function answers in days.
  next_birth_time = now + birth_speed(this group) days

  IF a percentile roll fails against birth_probability(this group) THEN RETURN

  born = round(current member count * uniform(0.5, 1.5) * birth_percentage / 100)
  IF born == 0 THEN RETURN
  create that many new members, each modelled on this group, and add them
```

**Invariants** — Growth is *proportional* to the existing population, so a group grows
geometrically and an extinct one never recovers. The uniform half-to-one-and-a-half jitter
on the count is what stops every pack in the world growing in lockstep.

The next birth time is advanced *before* the probability roll, not after, so a failed roll
still costs the group a full interval. Without that, the roll would be retried on every
update and the effective probability would be one.

**Notes** — The birth-speed function answers in days and is converted to the simulation's
millisecond clock by the literal chain 24 × 60 × 60 × 1000 — the only place in this file
where the game's calendar units surface. The related week-and-month units are in
[`alife_combat_manager.cpp`](alife_combat_manager.cpp.md).

New members are created by cloning from the group record rather than from a spawn file
entry, so a bred creature inherits the group's section and therefore all its tuning.

## `redundant` — when a corpse may be forgotten

**Contract** — Whether the simulation may discard this record entirely. Five conditions, all
of which must hold.

```text
FUNCTION redundant() -> bool
  IF alive                        THEN RETURN false
  IF online                       THEN RETURN false   # the player may be looking at it
  IF it has a story identifier    THEN RETURN false   # scripts name it; it must persist
  IF its death time is zero       THEN RETURN false   # it was authored dead: level dressing
  REQUIRE its death time is not in the future
  IF now < death time + configured stay-after-death interval THEN RETURN false
  RETURN true
```

**Invariants** — The zero death time is the sentinel set by
[`alife_creature_abstract.cpp`](alife_creature_abstract.cpp.md) for a creature spawned
already dead, and it means *never expire* — authored corpses are part of the level, not
garbage. That is why the sentinel is zero and not the clock reading: a real death always has
a non-zero timestamp, so the two cases are distinguishable forever.

A story identifier pins a record permanently. Scripts address named characters by that
identifier, and discarding one would make a quest unresolvable.

The assertion that death time is not in the future is a real integrity check on the save: a
loaded game whose clock went backwards relative to a recorded death would compute a negative
interval and expire nothing, or everything.

## `draw_level_position`

**Contract** — Where this creature should be drawn on the world map — taken from the brain's
movement manager rather than from the record's own position, because an offline creature's
position is a vertex plus a distance along an edge and the map wants an interpolated point.
