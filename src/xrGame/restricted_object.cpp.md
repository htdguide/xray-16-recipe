# src/xrGame/restricted_object.cpp

> Where a creature learns where it is allowed to go: builds its restrictor set at spawn, answers accessibility, and installs a temporary border around a path in progress.

**Needs** — [`restricted_object.h`](restricted_object.h.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`space_restriction.h`](space_restriction.h.md) · [`space_restriction_bridge.h`](space_restriction_bridge.h.md) · [`space_restriction_base.h`](space_restriction_base.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-query delegation, with one 4 KB string assembly at spawn

## Purpose

Every creature's movement space is the intersection of its restrictors: *out* restrictors
mark regions it must stay inside, *in* restrictors mark regions it must stay out of. This
file is the creature's end of that system. It decides three things: how a spawn record's
restrictors become a live set, what happens when that set changes, and how a creature
temporarily fences itself in for the duration of one path.

## State

```text
RECORD RestrictedObject
  subject : Creature
  applied : bool     # a temporary border is installed in the restriction manager
  removed : bool     # the border has been taken down; starts true
  actual  : bool     # the pathfinder's cached cost model matches the current set
```

**Invariants**

- `applied` and `removed` are **not** complements. `applied` means a border was actually
  installed; `removed` means the install/remove bracket is currently closed. A border
  request from an *inaccessible* start position installs nothing but still opens the
  bracket, so both are false at once. The bracket must balance: an install asserts it is
  closed, a removal asserts it is open.
- The restriction *sets* themselves live nowhere in this record. They are held by the
  level's restriction manager keyed by entity identifier, which is what lets an offline
  entity keep its restrictors and what makes every query here a delegation.
- **Every mutation of the set invalidates `actual`, and invalidating it notifies the
  creature.** That notification is the entire reason the flag exists: the pathfinder caches
  a cost model per creature, and a changed restrictor set silently invalidates it. See
  `actual` below.

## `net_Spawn`

**Contract** — builds the creature's restriction set from its server record and installs it.
Runs once, as part of the creature's spawn, before any query can be asked. Requires the
server record to be a creature record. Resets the border bracket to closed, and marks the
cost model current.

```text
FUNCTION net_Spawn(server_record)
  applied = false; removed = true

  out_names = server_record.static_out_restrictors     # authored, a comma-separated
  in_names  = server_record.static_in_restrictors      #   list of restrictor names

  IF the alife simulation is running THEN
    append_names_of(server_record.dynamic_out_restrictions, TO out_names)
    append_names_of(server_record.dynamic_in_restrictions,  TO in_names)

  restriction_manager.restrict(server_record.id, out_names, in_names)
  actual = true
```

**Invariants**

- The set has two origins that must be merged here and nowhere else: **static** restrictors
  named in the authored spawn record as text, and **dynamic** restrictors held as entity
  identifiers, which the alife simulation and scripts attach at run time. The merge
  direction is one-way — identifiers are resolved to names and the whole thing becomes a
  name string, because the restriction manager's vocabulary is names.
- Dynamic restrictors are resolved **only when the alife simulation is running**, and a
  resolved restrictor is skipped unless its server record sits on the *current level*. A
  restrictor on another level cannot constrain anything here, and naming it would ask the
  manager to look up a restrictor that does not exist locally.
- The merge builds into a fixed 4 KB buffer. That is a real ceiling on how many restrictors
  one creature can carry; a rebuild should use a growable list of names and skip the string
  entirely.

**Notes** — the name-string representation is the awkward part of this system: identifiers
are turned into names so they can be concatenated, then the manager parses them back apart.
A rebuild should pass the set as a list of restrictor references throughout and treat the
comma-separated form purely as the authored input format.

## `net_Destroy`

**Contract** — drops this creature's entire restriction set from the manager. Note it reads
the creature's identifier directly rather than through the checked accessor, which is the
only place it does so — by destruction time the creature may already be partly torn down.

## `accessible` — the four forms

**Contract** — "may this creature be here?". Delegates to the restriction manager, which
intersects the creature's out-restrictors and complements its in-restrictors. Four shapes:

- a position, tested as a sphere of a negligible radius;
- a position with an explicit radius, tested as a sphere;
- a navigation vertex, likewise with a negligible radius;
- a navigation vertex with a radius.

The vertex forms require a valid vertex. None allocate; all block for the duration of the
manager's test, which is the hot path of pathfinding and is profiled as such.

**Notes** — the degenerate-radius default is a small epsilon rather than zero, so that a
position exactly on a restrictor boundary reads as *inside* the restricted region. A rebuild
choosing zero will let creatures stand precisely on forbidden edges.

## `accessible_nearest`

**Contract** — given an inaccessible position, answers the nearest position this creature
*is* allowed to occupy, and the navigation vertex it belongs to. **Requires the input to be
inaccessible** — asserting that is the contract, not a sanity check, because the manager's
search is defined only for a point outside the permitted space. Callers test first.

## `add_border` — the three forms

**Contract** — installs a temporary extra restriction for the duration of one path, so that
a creature walking somewhere cannot wander off the route. Three shapes, by how the route is
described: a start vertex plus a radius, a start and destination position, or a start and
destination vertex. Asserts the bracket is closed and opens it.

```text
FUNCTION add_border(route description)
  ASSERT NOT applied AND removed
  removed = false                       # the bracket opens whether or not a border lands
  IF accessible(start) THEN
    applied = true
    restriction_manager.add_border(subject.id, route description)
```

**Invariants** — a border is installed **only if the start is already accessible**. A
creature that is standing somewhere it should not be gets no border, because fencing it into
a forbidden region would leave it with nowhere legal to go. The bracket still opens, which is
why `applied` and `removed` are separate flags.

## `remove_border`

**Contract** — closes the bracket and removes the border if one was installed. Asserts the
bracket is open. Idempotence is not offered: a second removal asserts.

## `add_restrictions` / `remove_restrictions` — identifier-list forms

**Contract** — add or withdraw restrictors named by entity identifier at run time. Each
identifier is resolved to a live object; an unresolvable one is skipped, and so is one that
is **already in the desired state** — already present when adding, already absent when
removing. For each one that actually changes, a network event carrying the identifier and
the restrictor type is sent, *and* the name is accumulated into a string handed to the
restriction manager. Empty inputs return immediately. Marks the cost model stale.

**Invariants** — the redundancy check is a substring match of the restrictor's name against
the creature's current restriction string. That is a textual test, so a restrictor whose
name is a substring of another restrictor's name is wrongly considered already present. A
rebuild holding the set as a set rather than as a string does not have this problem, and
should not reproduce it.

**Notes** — the event and the local change are both performed, so in single player the
change is applied twice: once directly and once when the creature handles its own event.
That is the general shape of the engine's server/client split — the local application is the
optimistic one — but here it means the manager must tolerate a redundant add.

## `add_restrictions` / `remove_restrictions` — name-string forms

**Contract** — the same, taking an already-assembled comma-separated name string. No
per-restrictor event is sent and no redundancy filtering happens: the string goes straight to
the manager. This is the path the script layer and the alife simulation use. Marks the cost
model stale.

## `remove_all_restrictions`

**Contract** — clears everything. Sends a remove-all event for the out type and another for
the in type, marks the cost model stale if anything was actually set, and then resets the
creature's set in the manager to empty. The order matters: the events describe the change to
anyone replaying it, and the direct reset makes it true locally.

## `actual`

**Contract** — the setter, and the only non-trivial one in the file. Records whether the
cached cost model is current, and **on a transition to stale, notifies the creature**.

```text
FUNCTION set_actual(value)
  actual = value
  IF NOT actual THEN subject.on_restrictions_change()
```

**Invariants** — this is the hinge of the whole file. The pathfinder builds a per-creature
cost model that bakes in which vertices are reachable; a changed restrictor set silently
invalidates it, and a creature that keeps pathing against a stale model will walk into a
region it is no longer allowed in, or refuse a region it now may enter. Every mutator in
this file ends by marking it stale, without exception, and a rebuild that adds a mutator
must do the same.

**Notes** — the notification fires on every set-to-stale, including when it was already
stale. Harmless because the creature's handler is idempotent, but it means the count of
notifications is not the count of changes.

## `in_restrictions` / `out_restrictions` / `base_in_restrictions` / `base_out_restrictions`

**Contract** — read the creature's current restrictor names, and its spawn-time ones, as
comma-separated strings. The base pair is what the spawn record asked for; the plain pair is
what is in force now. Scripts use the difference to restore a creature to its authored
constraints.
