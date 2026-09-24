# src/xrGame/alife_group_abstract.cpp

> A squad that exists offline as one record and online as several creatures: this is the expansion and the collapse, plus the rule that a member who dies leaves the group and becomes an individual.

**Needs** — [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`xrAICore/Navigation/game_level_cross_table.h`](../xrAICore/Navigation/game_level_cross_table.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registry bookkeeping plus one placement computation.

## Purpose

Simulating twelve creatures walking across the world off-screen costs twelve times what
simulating one costs, and the player cannot tell the difference. So a group is *one* record
while offline — one position, one speed, one path along the graph — and becomes its members
only when the player gets close. This file is that transformation in both directions.

## State

```text
RECORD GroupAbstract                     # mixed into a dynamic server object
  members                 : list<entity id>
  count                   : int          # member count, maintained alongside the list
  create_spawn_positions  : bool         # have the members never been placed yet?
  # as a creature record it also carries: position, level vertex, graph vertex,
  # previous and next graph vertex, distance along the edge, current speed

# Invariant: an empty group is redundant — the simulation may discard it.
# Invariant: while offline, the members' own positions are meaningless; the group's
#   position is the truth. While online, the reverse.
```

## `switch_online` — expansion

**Contract** — Turns the record into its members. Asserts the group was offline. Places
every member, brings each online, and takes the group itself out of the offline registries.

```text
FUNCTION switch_online()
  REQUIRE offline
  online = true
  FOR EACH member, by index i out of n
    IF this is the group's first ever expansion
      member.position     = group.position
      member.level_vertex = group.level_vertex
      IF the member is a creature
        member.torso_yaw = normalize(i / n * full turn)
    IF the member is not already online
      bring it online, without touching the registries

  first expansion = false
  remove the group from the coarse scheduler
  remove the group from the graph registry
```

**Invariants** — The members are stacked on the group's exact position, differing only in
which way each faces — spread evenly around the compass. They are not scattered, because
there is no guarantee any nearby position is walkable; the engine relies on the physics and
navigation layers to push overlapping creatures apart on the first frames. The compass
spread is what stops them all facing the same way and looking like a formation.

The placement happens only on the *first* expansion. Afterwards the members keep their own
positions across collapse and re-expansion — because the collapse records the group's
position from a member, so the information is preserved on the other side.

**Notes** — The angle computation divides an integer index by an integer count, which in the
original's arithmetic truncates to zero for every member but none — so every member in fact
faces the same way. That is a bug, not a decision; a rebuild should divide in real
arithmetic and get the intended fan.

## `switch_offline` — collapse

**Contract** — Turns the members back into one record. Asserts the group was online. Adopts
the first member's position as the group's, re-derives the group's graph placement from it,
picks a random previous vertex, sends every member offline, and puts the group back into the
offline registries.

```text
FUNCTION switch_offline()
  REQUIRE online
  online = false
  IF there is a first member
    group.speed        = its configured cross-level travel speed
    group.position     = first member's position
    group.graph_vertex = cross_table[group.level_vertex].game_vertex
    group.distance_to_next = cross_table[group.level_vertex].distance
    group.next_vertex  = group.graph_vertex
    group.previous_vertex = a uniformly random neighbour of group.graph_vertex
    take the first member offline, without touching the registries
  FOR EACH remaining member
    take it offline, without touching the registries
  add the group to the coarse scheduler
  add the group to the graph registry
```

**Invariants** — The group's offline travel speed is *not* the speed its members were moving
at; it is a separate configured "going between levels" speed. Off-screen travel is not a
scaled version of on-screen walking, it is its own model.

The previous vertex is chosen at random from the current vertex's neighbours because the
group has to be *somewhere on an edge* to be travelling, and where it actually came from is
not recorded. That randomness is visible in gameplay: a group that goes offline and
immediately reverses may appear to have come from a direction it did not.

The group's graph vertex is re-derived from its *level* vertex through the cross table
rather than copied from the member, which is the right direction — the level vertex is the
fine truth and the graph vertex is derived from it everywhere in the engine.

## `synchronize_location`

**Contract** — Pushes every member's position through its own synchronization, then adopts
one member's placement as the group's.

**Notes** — As written, the member whose placement is adopted is read *after* the iteration
has run off the end of the member list. That is an out-of-bounds read: the intended member
is almost certainly the first. It is reached only for a non-empty group whose members have
just been synchronized, so in practice it reads whatever lies past the list. A rebuild must
pick a member deliberately — the first is the one the collapse also uses, so the first is
the consistent choice.

The rest of the routine is the same online/offline asymmetry as in
[`alife_dynamic_object.cpp`](alife_dynamic_object.cpp.md): an offline group changing graph
vertex is a registry move, an online one is a field assignment.

## `try_switch_online`

**Contract** — An empty group never comes online — there would be nothing to spawn.
Otherwise the ordinary dynamic-object promotion decides.

## `try_switch_offline` — collapse eligibility and the death rule

**Contract** — A group may collapse only when *every* living member is individually ready to
collapse. Walking the members to find out, it also performs the death rule: a dead member is
removed from the group and promoted to a standalone object.

```text
FUNCTION try_switch_offline()
  IF the group is empty THEN RETURN
  FOR EACH member (by index, the list is mutated during the walk)
    IF the member is not a creature THEN CONTINUE
    IF the member is alive
      IF it cannot go offline          THEN CONTINUE      # it does not block the group
      IF it cannot go online           THEN BREAK         # it must be offline: block
      IF it is still within the offline distance of the player THEN BREAK
      CONTINUE
    # the member is dead: detach it from the group entirely
    set its health to zero
    hand it to direct simulation control
    erase it from the member list; decrement the member count
    IF it is an item held by somebody THEN detach it from that owner
    register it with the simulation as an independent object
    IF it is not a held item
      remove it from the graph registry, keeping it on the current level
    leave it marked online

  IF the group is now empty THEN RETURN
  IF the group itself cannot go offline THEN RETURN
  IF the group can go online, or the walk completed without blocking
    collapse the group
```

**Invariants** — The loop's break condition is the whole policy: one member who is still
near the player, or who must stay online, keeps the entire group expanded. A group does not
partially collapse.

A member that *cannot* go offline at all does not block — it continues — which is the
opposite of a member that cannot go *online*. The first is something like a corpse that is
fine either way; the second is something the engine requires to be a record, so the group
must wait.

The dead member is removed and left **online**. It becomes a body lying on the level,
independently registered, no longer the group's concern. Its removal from the graph registry
is explicitly told not to remove it from the current level, because it is still there —
only its group membership ended.

The loop mutates the list it walks and compensates by stepping the index and the bound back
together. A rebuild should walk a snapshot or partition instead; the compensation is correct
here but is exactly the pattern that breaks under later edits.

The final condition — *can go online, or the walk completed* — is the escape hatch for a
group all of whose members are dead or unblocking: if the walk never broke, everyone is
ready.

## `redundant`

**Contract** — A group with no members is redundant and may be discarded by the simulation.
This is how groups disappear when their last member dies or leaves.
