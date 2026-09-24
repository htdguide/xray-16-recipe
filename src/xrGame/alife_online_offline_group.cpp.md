# src/xrGame/alife_online_offline_group.cpp

> The squad: a server object that absorbs a set of creatures so the whole group travels the world as one record while offline, and dissolves into individually simulated creatures while online.

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`alife_online_offline_group_brain.h`](alife_online_offline_group_brain.h.md) · [`alife_monster_movement_manager.h`](alife_monster_movement_manager.h.md) · [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_level_cross_table.h`](../xrAICore/Navigation/game_level_cross_table.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registry bookkeeping and a distance test per member

## Purpose

The alife simulation cannot afford to path a dozen creatures individually across the whole
world map, and the game does not want it to: a squad of bandits should arrive together,
not trickle in. This class is the answer. A **group** is a first-class server object that
owns a set of creature records; while the squad is offline the group is the thing that
exists — it holds one position, it is the one entity on the game graph, it is the one
entity the scheduler updates, and it pushes its position down onto every member. While
the squad is online, the members are real creatures walking a level and the group becomes
their shadow, copying its position *up* from whichever member is the commander.

The whole file is the machinery of that inversion, plus the rules for when the flip may
happen. It is the densest piece of registry bookkeeping in the alife layer and the
easiest place to leave an entity registered twice.

## State

```text
RECORD OnlineOfflineGroup            # extends the abstract alife creature record
  members        : map<EntityId, ref MonsterRecord>   # ordered; the first is the commander
  online         : bool
  position       : vector
  level_vertex   : int
  graph_vertex   : int                # invalid until the first member joins
  distance       : real               # travel distance accumulated; mirrored onto members
  going_speed    : real               # copied from the first member at formation
  level_speed    : real               # ditto
  terrain_mask   : list<TerrainType>  # where this squad may walk; mirrored onto members
  uses_ai_locations : flag            # false while empty, true once populated
```

Invariants that the rest of the file exists to maintain:

- **An entity is registered in the graph registry and the scheduler exactly once.** Either
  the group is registered and its members are not (offline), or the members are and the
  group is not (online). Every transition below is a careful hand-off between those two
  arrangements.
- **Every member's position equals the group's** while offline; the group's position
  equals the commander's while online. The copy direction flips with the online flag and
  nothing else.
- A member's back-reference to its group is the group's identifier while joined and the
  invalid identifier while free. It is set and cleared in exactly one place each.
- The commander is simply the first member in identifier order. There is no election —
  the "commander" is a position in an ordered container, which means a squad's leader
  changes silently when a lower-numbered member joins or the current leader dies. A
  rebuild reproducing the original's behaviour must keep that ordering.
- An empty group is **redundant**: it is not saved and the simulation is free to delete
  it. This is how squads disappear when their last member dies.

## `update`

**Contract** — the group's per-tick alife update. Cheap; no allocation.

```text
FUNCTION update()
  IF online
    commander = members.first
    position, level_vertex, graph_vertex = commander's          # copy up
  IF NOT active                                # active = offline AND has members
    RETURN
  brain.update()                               # decide and move, offline only
  FOR EACH member IN members
    member.position, member.level_vertex,
    member.graph_vertex, member.distance = group's              # copy down
```

**Invariants** — the copy-up happens *before* the activity test, so an online group's
record stays current even though its brain never runs; that is what lets the group be
saved, be found by a script, and be switched offline at a sensible position at any moment.
The copy-down happens *after* the brain, so members reflect the move decided this tick
rather than the previous one.

`active` is exactly "offline and non-empty". An online group has no decisions to make
(its members are making their own), and an empty group has nothing to decide for.

## `register_member`

**Contract** — absorbs an existing creature record into the group. The creature must
exist, be a creature, be alive, and not already be in this group. Mutates three registries.
This is the single most order-sensitive routine in the file.

```text
FUNCTION register_member(member_id)
  REQUIRE member_id not already in members
  member = alife.objects.lookup(member_id)
  REQUIRE member is a creature record AND member is alive

  was_empty = members.is_empty()

  # Bring the member's registrations into line with the group's online state.
  IF member is offline
    IF group is online
      member.switch_online()                  # promote it to match the group
      REQUIRE member has no parent            # a carried creature cannot be a squad member
      alife.graph.level.remove(member)        # the group's members are not level-registered
    ELSE
      alife.graph.remove(member, member.graph_vertex)  # the group holds the graph slot now
      alife.scheduled.remove(member)                   # and the schedule slot
  ELSE                                         # member is online
    IF group is offline
      group.switch_online()                    # an online member forces the group online
    REQUIRE member has no parent
    alife.graph.level.remove(member)

  REQUIRE member.group_id is unset OR already this group
  member.group_id = this.id
  members.insert(member_id, member)

  IF NOT was_empty -> RETURN

  # The first member defines the group.
  position, level_vertex, graph_vertex = member's
  going_speed, level_speed             = member's
  uses_ai_locations = true
  alife.graph.update(this)                     # now the group takes a graph slot
```

**Invariants** —

- Whichever of the two is already online wins: an offline member joining an online group
  is promoted; an online member joining an offline group drags the group online. The
  group and its members are never in mixed states, which is the precondition every other
  routine here assumes.
- Deregistration always precedes registration of the replacement holder, never the other
  way round, so the "registered exactly once" invariant is never transiently violated in
  the direction that would trip the duplicate check.
- The first member seeds the group's position *and* its two speeds. The speeds are never
  recomputed as the squad's composition changes, so a squad travels at the speed of
  whoever founded it, not the speed of its slowest member. That is a real gameplay
  decision — squads do not slow down when a wounded member joins — and it looks like an
  oversight until you notice the field is never touched again.
- `uses_ai_locations` is raised only when the group becomes non-empty, because a group
  with no position must not be placed on the navigation graph at all.

## `unregister_member`

**Contract** — releases a member back to independent existence. The member must currently
belong to this group.

```text
FUNCTION unregister_member(member_id)
  member = members.lookup(member_id)      # REQUIRE present and owned by this group
  member.group_id = unset
  alife.graph.update(member)              # the member reclaims its own graph slot
  alife.scheduled.add(member)             # and its own schedule slot
  members.remove(member_id)
  IF members.is_empty()
    uses_ai_locations = false
```

**Invariants** — the back-reference is cleared *before* the registries are touched,
because the graph update consults it. The group does not surrender its own registrations
when it empties; it merely stops claiming navigation locations and becomes redundant, and
the simulation reaps it separately.

**Notes** — this is symmetric with the offline half of `register_member` only. A member
leaving an *online* group is not re-added to the level registry here, because an online
member is already a live client object and the level registry entry is recreated by the
switch machinery. The asymmetry is genuine and easy to get wrong in a rebuild.

## `notify_on_member_death`

**Contract** — a member died; drop it from the squad. Delegates straight to
`unregister_member`. A dead creature is never a squad member — the group's brain assumes
every member is alive and asserts it — so the death hook is the only thing keeping that
assumption true.

## `switch_online`

**Contract** — the group becomes a set of live creatures. Fails hard if already online.

```text
FUNCTION switch_online()
  REQUIRE NOT online
  online = true
  FOR EACH member IN members
    IF member is offline
      alife.add_online(member, update_registries = false)
  alife.scheduled.remove(this)              # the group stops being updated as one
  alife.graph.remove(this, graph_vertex, update_registries = false)
```

**Invariants** — members first, then the group's own deregistration. A member coming
online may consult the group (it still has a group identifier), so the group must remain
in a coherent registered state until every member is up.

The "do not update registries" flag on both calls is the crux: the member's own promotion
must not perform the usual level-registry bookkeeping, because the group is managing
those slots by hand. A rebuild that hides this behind a single "promote" operation must
still expose that distinction, or the same entity ends up registered by two owners.

## `switch_offline`

**Contract** — the live creatures collapse back into one travelling record. Fails hard if
already offline.

```text
FUNCTION switch_offline()
  REQUIRE online
  online = false
  IF members not empty
    commander = members.first
    commander.synchronize_location()        # force its record to match its live position
    position, level_vertex, graph_vertex, distance = commander's
  FOR EACH member IN members
    IF member is online
      member.clear_client_data()            # drop the live-instance state before demoting
      alife.remove_online(member, update_registries = false)
  alife.scheduled.add(this)
  alife.graph.add(this, graph_vertex, update_registries = false)
```

**Invariants** — the commander's location is harvested **before** any member is demoted,
because demotion destroys the live instance the position would have come from. Then every
member goes down, and only then does the group take up its own graph and schedule slots —
the mirror image of the online transition, and for the same reason.

Clearing a member's client data before demotion is what makes the round trip lossless
(conformance criterion 11): the live instance's transient state is discarded explicitly
rather than left to be reinterpreted as record state on the way back up.

## `try_switch_online`

**Contract** — asked periodically by the switch manager. Decides whether *this* group
should be promoted, using the whole squad rather than the group's own position.

```text
FUNCTION try_switch_online()
  IF members.is_empty()        -> RETURN
  IF NOT can_switch_online()   -> RETURN
  IF NOT can_switch_offline()  -> promote unconditionally; RETURN   # pinned online
  FOR EACH member IN members
    IF distance(actor.position, member.position) <= alife.online_distance
      promote; RETURN
  on_failed_switch_online()
```

**Invariants** — **any one member within the online radius promotes the entire squad.**
That is the point of the class: a squad arrives whole. Testing the group's own position
would be wrong, because the group's position is the commander's and a trailing member
could otherwise pop in from nothing.

An entity that *cannot be switched offline* is one that must always be online; such a
group is promoted without any distance test, since there is no state in which it may
legitimately stay down.

## `try_switch_offline`

**Contract** — the mirror decision, with the opposite quantifier.

```text
FUNCTION try_switch_offline()
  IF members.is_empty()        -> RETURN
  IF NOT can_switch_offline()  -> RETURN
  IF NOT can_switch_online()   -> demote unconditionally; RETURN    # pinned offline
  FOR EACH member IN members
    IF distance(actor.position, member.position) <= alife.offline_distance
      RETURN                  # someone is still close; the squad stays up
  demote
```

**Invariants** — **every** member must be beyond the offline radius before the squad goes
down, where **any** member inside the online radius brings it up. The asymmetry, together
with the fact that the offline radius is larger than the online radius, is what prevents a
squad straddling the boundary from flipping every tick. A rebuild that uses one radius, or
the same quantifier on both sides, will produce visible thrashing on the shipped data.

## `on_failed_switch_online`

**Contract** — runs when a promotion was considered and declined. Clears every member's
live-instance state.

**Notes** — this looks redundant (the members are offline; there is no live state) and is
not. A member may carry stale client data from a *previous* online period; leaving it
there means the next promotion resurrects an old pose, an old animation state or an old
damage record. The failed attempt is simply the convenient moment to clean up.

## `synchronize_location`

**Contract** — pulls position, vertices and travelled distance up from the commander when
online; a no-op when offline. Always reports success. This is the hook the save path and
the switch machinery call when they need the group's record to be exactly current rather
than a tick stale.

## `force_change_position`

**Contract** — teleport: places the group at an arbitrary world position, re-deriving both
navigation vertices and moving the group's game-graph registration if the coarse vertex
changed.

```text
FUNCTION force_change_position(position)
  new_level_vertex = level_graph.vertex_containing(position)
  new_graph_vertex = cross_table.lookup(new_level_vertex).game_vertex
  this.position     = position
  this.level_vertex = new_level_vertex
  IF graph_vertex != new_graph_vertex
    alife.graph.change(this, graph_vertex, new_graph_vertex)
```

**Invariants** — the coarse vertex is derived from the fine one through the level's
**cross table**, never computed from the position directly; that table is the authority
on which game-graph vertex a level position belongs to, and deriving it any other way
produces a squad registered under a vertex the pathfinder disagrees with.

The group's own coarse-vertex field is updated by the registry move, not here — the
registry needs the old value to find the entry it is moving. A rebuild passing the new
value must not have already overwritten the old one.

This requires a loaded level graph, so it is only callable for a group on the current
level.

## `clear_location_types` / `add_location_type`

**Contract** — replace or extend the terrain mask that says where this squad may walk.
Both apply the change to the group **and** to every current member.

**Invariants** — the mirroring is necessary because the group paths on the game graph
using its own mask while offline, and each member paths on its own mask once online; a
squad whose members could walk somewhere the group could not (or the reverse) would
teleport or get stuck at the transition. Note that a member joining *after* a mask change
does not inherit it — masks are pushed, never pulled — which is a real difference in
behaviour depending on the order of scripted calls.

`add_location_type` takes an authored mask string and merges it into the existing terrain
list rather than replacing it.

## `on_before_register`

**Contract** — runs before the group enters any registry. Sets the coarse vertex to the
invalid value and lowers `uses_ai_locations`.

**Invariants** — a freshly created group has no position, because its position comes from
its first member. Declaring the vertex invalid and disclaiming navigation locations is
what stops the registration machinery from trying to place a positionless entity on the
graph. This is the one place where "a group is not yet anywhere" is expressible.

## `on_after_game_load`

**Contract** — rebuilds every membership after a save is restored.

```text
FUNCTION on_after_game_load()
  IF members.is_empty() -> RETURN
  ids = the member identifiers                 # only the keys survived the save
  members.clear()
  FOR EACH id IN ids
    register_member(id)
```

**Invariants** — a save stores membership as a set of **identifiers with no resolved
references**; the group's own load leaves the map populated with keys and empty values.
Re-registering each identifier is what resolves them, and it also re-runs every registry
hand-off, so a loaded squad ends up in exactly the arrangement a live one would be. That
is why the routine cannot simply look each identifier up and fill in the value.

Ordering requirement this imposes on the rest of the load: **every member record must
already be in the object registry when this runs.** The registry load is flat and
ordered by identifier, and a group's identifier is not guaranteed to be lower than its
members', so this hook necessarily runs in a later pass over all objects — it cannot be
folded into the group's own deserialization.

## Group-level queries

**Contract** —

- `commander_id` — the first member's identifier, or the invalid identifier when empty.
- `squad_members` — the whole membership table.
- `npc_count` — how many members.
- `member(id)` — one member by identifier; absence is a diagnosed fault unless the caller
  says it is tolerable.
- `active` — offline and non-empty; the condition under which the brain runs.
- `redundant` — empty; the condition under which the simulation may discard the group.

## Inherited behaviours the group explicitly declines

**Contract** — three creature behaviours are answered with a fixed value rather than
implemented:

- *best weapon* — none. A group does not carry equipment; its members do.
- *best detector* — none, for the same reason.
- *meeting action type* — always *ignore*. Offline encounters between a group and
  another schedulable entity produce no fight, no trade and no reaction.

**Notes** — these are not stubs. The offline encounter system was designed for individual
creatures, and a squad deliberately opts out of it: whatever a squad does on meeting
someone is decided by its brain and its scripts, not by the generic offline-meeting rules.
A rebuild should keep the opt-out explicit rather than inheriting the creature behaviour
and hoping it does nothing.

`need_update` is likewise always true: a group is never skipped by the scheduler on the
grounds of having nothing to do.
