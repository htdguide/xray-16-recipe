# src/xrGame/Level_network_spawn.cpp

> Turns a spawn record into a live object: decodes the record off the wire, asks the factory for a client object, runs the spawn lifecycle, and wires up ownership and player control.

**Needs** — [`Level.h`](Level.h.md) · [`xrServer_Objects_ALife_All.h`](../xrServerEntities/xrServer_Objects_ALife_All.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`NET_Queue.h`](NET_Queue.h.md) · [`client_spawn_manager.h`](client_spawn_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_level_cross_table.h`](../xrAICore/Navigation/game_level_cross_table.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a lifecycle whose ORDER is the content, over a factory keyed by a data-supplied class identifier

## Purpose

Everything that exists in a level got here. A spawn record — the authored payload from the
level's spawn file, or the same structure sent by a server, or one synthesized at runtime —
names a class and carries its state. This file is where that record becomes a client
object registered with the scheduler, the renderer and the physics world.

The order of the steps is the file's entire value and it is load-bearing: create, read the
record, reject on a configuration mismatch, create the client object from the factory, run
its spawn against the record, and only then fire the callbacks, claim player control and
issue the ownership event. A rebuild that reorders any of these produces objects that are
half-registered when something first looks at them.

The server record is transient in all of these paths. It is decoded, used to build the
client object, and destroyed immediately. The client object keeps whatever it needs.

## State

`Stateless` beyond the deferred spawn queue it drains, which lives on the level.

## `cl_Process_Spawn`

**Contract** — decodes a spawn message and spawns the entity it describes. Fails hard if the
class name has no registered factory entry. Silently discards the record when the entity
declares itself inapplicable to this game configuration. Allocates a server record and
frees it before returning.

```text
FUNCTION cl_Process_Spawn(message)
  class_name = read length-prefixed string
  record = entity_factory.create(class_name)      # FAIL WITH unknown class if absent
  record.read_spawn(message)
  IF record.flags has SPAWN_UPDATE THEN record.read_update(message)

  # An entity can veto its own existence based on the current game type, difficulty or
  # ruleset. This is how one spawn file serves several game modes.
  IF NOT record.matches_configuration() THEN
    destroy record
    RETURN
  END IF

  # On the authoritative side every spawned object is local by definition.
  IF running as the server THEN record.flags += SPAWN_OBJECT_LOCAL

  g_sv_Spawn(record)
  destroy record
```

**Notes** — the spawn record carrying an optional *update* payload in the same message is
the mechanism by which an entity arrives already in a non-default state: an alife entity
promoted to online brings its accumulated health, ammunition and position with it in one
message rather than in a spawn followed by an update.

## `g_sv_Spawn`

**Contract** — the lifecycle. Given a decoded server record, creates the client object,
runs its spawn, and on success fires the pending-callback hook, claims player control if
the record asks for it, and issues the ownership event that places the object into its
parent's inventory. On failure, unwinds fully. Notifies the game rules at the end.

**Invariants** — a failed spawn must leave nothing behind: the partially constructed object
is destroyed, its pending script callbacks are cleared, and it is removed from the object
registry. Ownership is issued *after* the object is fully spawned, never before, because
the receiving container will immediately query the item.

```text
FUNCTION g_sv_Spawn(record)
  # Single player is one process, so the update-minimization flag is safe; multiplayer
  # needs the full update stream.
  network minimize-updates = (single player)

  object = object_registry.create(record.class_name)
  IF object is none OR NOT object.net_Spawn(record) THEN
    object.net_Destroy()
    clear this object's pending client-spawn callbacks
    object_registry.destroy(object)
    log the failure
    RETURN
  END IF

  fire this object's pending client-spawn callbacks     # scripts waiting on this id

  IF record.flags has LOCAL and ASPLAYER THEN
    IF a demo is playing THEN
      IF record.flags has PHANTOM THEN                  # the fake demo spectator
        control entity = object; view entity = object; demo spectator = object
      END IF
    ELSE
      IF there is a current entity THEN tell it it is no longer the current entity
      control entity = object; view entity = object
    END IF
  END IF

  IF record.parent_id is set THEN
    # The object was spawned already owned. Deliver the ownership event IMMEDIATELY
    # rather than enqueueing it, so the parent's inventory is consistent before any
    # other message touches either object.
    deliver OWNERSHIP_TAKE(parent: record.parent_id, child: object.id)
  END IF

  game rules.on_spawn(object)
```

**Notes** — the ownership event is built and dispatched inline instead of being pushed onto
the game-event queue, and the original carries the enqueueing version commented out beside
it. The difference is visible: enqueued, an item spawns as a loose world object and is
picked up a frame later, which flickers. Delivered inline, it is never loose. A rebuild
should keep the inline delivery and accept that it makes spawn re-entrant with respect to
event dispatch.

Claiming control marks the *previous* current entity first. That notification is what makes
the old actor put its first-person weapon model away; skipping it leaves two rendered
first-person views.

The demo case deliberately claims both the control entity and the view entity from a
phantom spectator, and only while a demo is running — a real player-flagged spawn during
playback must not steal the camera.

## `g_cl_Spawn`

**Contract** — asks the server to spawn an entity by class name at a respawn point and a
position. Builds a fresh record with unassigned identifiers, writes it as a spawn message,
sends it reliably, and destroys the record. Does not create anything locally — the object
comes back as an ordinary spawn message.

**Invariants** — the entity, parent and phantom identifiers are all left at the reserved
"none" value; the server assigns them. Writing a real identifier here would collide.

**Notes** — this is the client's only way to create anything. Even on a listen server the
request goes through the message path, so that the authoritative side allocates the
identifier. That single rule is what keeps the entity-identifier space consistent.

## `spawn_item`

**Contract** — builds a spawn record for a configuration section at a position, optionally
snapped to a navigation vertex and optionally parented. With `return_item` false it sends
the record and returns nothing; with it true it returns the record for the caller to fill
in further and send itself. Fails hard if the section names no known class.

```text
FUNCTION spawn_item(section, position, level_vertex, parent_id, return_item) -> record or none
  record = entity_factory.create(section)      # FAIL WITH unknown section

  IF record is an alife dynamic object AND a level graph is loaded THEN
    record.level_vertex = level_vertex
    IF level_vertex is valid AND the game graph and cross table are loaded THEN
      record.game_vertex = cross_table.lookup(level_vertex).game_vertex
    END IF
  END IF

  # A spawned weapon arrives with a full magazine. Anything else would require the
  # caller to know each weapon's capacity.
  IF record is a weapon THEN record.rounds_loaded = record.magazine_size

  record.class_name = section; record.display_name = section
  record.position = position
  record.respawn_point = NONE
  record.id = NONE; record.parent_id = parent_id; record.phantom_id = NONE
  record.flags = SPAWN_OBJECT_LOCAL
  record.respawn_time = 0

  IF return_item THEN RETURN record
  send record as a spawn message, reliably
  destroy record
  RETURN none
```

**Invariants** — deriving the coarse game-graph vertex from the fine level vertex through
the cross table, rather than searching for it, is required: an alife entity's position is a
graph vertex plus an offset, and the two graphs must agree or the entity teleports when it
goes offline.

**Notes** — this is the entry point the script layer's item-creation functions land on, so
its defaults *are* the defaults modders see. The full magazine in particular is a
gameplay-visible decision, not a convenience.

## `ProcessGameSpawns`

**Contract** — drains the deferred spawn queue, spawning each record and destroying it.
Called at the defined point in the frame where world-changing messages are applied.

**Notes** — the queue is only populated by a code path that the original leaves commented
out; in the shipping configuration spawns are processed inline from the message dispatch
and this drains an empty queue. It is kept because the deferred path is the correct one for
a rebuild that wants spawns batched at one point in the frame, and the inline path is the
one that avoids a frame of latency. Both cannot be true at once; the engine chose latency.
