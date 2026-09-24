# src/xrGame/game_sv_item_respawner.cpp

> Keeps the pickup points stocked: every point holds a prototype entity, and when the item on it is taken a timer starts that clones the prototype back into the world.

**Needs** — [`game_sv_item_respawner.h`](game_sv_item_respawner.h.md) · [`game_sv_base.h`](game_sv_base.h.md) · [`Level.h`](Level.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md)
**Used by** — [`game_sv_item_respawner.h`](game_sv_item_respawner.h.md)
**Tier floor** — T2: serializes a prototype entity into a spawn packet and replays it

## Purpose

Multiplayer maps are stocked with weapons and ammunition that reappear a fixed time after
being taken. The mechanism is the interesting part: rather than re-reading configuration on
each respawn, each pickup point holds a **fully constructed prototype entity** that is never
itself put into the world. Respawning means serializing the prototype into a spawn packet and
feeding that packet back through the server's normal spawn path — so a respawned item is
created by exactly the same code as an originally spawned one, and the prototype's tuning
(attachments, magazine contents, position, orientation) is applied once at load and reused
forever.

## State

```text
RECORD Pickup                     # one cycling spawn point
  prototype        : server_object   # never registered in the world; a template
  respawn_delay    : int             # milliseconds
  current_instance : int             # entity id of the live copy; all-ones when none
  taken_at         : int             # server time it was taken; 0 means "not waiting"

RECORD LoadoutRow                 # one line of the respawn configuration
  section  : text
  delay    : int                  # authored in seconds, stored in milliseconds
  addons   : int (8-bit)          # attachment flag set
  ammo     : int (16-bit)         # rounds to load

RECORD Respawner
  pickups        : list<Pickup>
  loadout_cache  : map<text, list<LoadoutRow>>   # profile name to its rows
  level_items    : set<int>                      # loose items from the round spawn file
  packet         : spawn_packet                  # one reused scratch buffer
```

**Invariants**

- **A taken time of zero means "not counting down"**, and is how the update loop distinguishes
  a pickup that is currently stocked from one that is waiting. It is set when the item is
  taken and cleared when it is respawned. A pickup whose item has never been taken is
  therefore indistinguishable from one currently stocked, which is correct.
- The prototype is owned by the pickup and destroyed with it. It is never registered with the
  server, never has an identifier, and is never seen by a client.
- The loadout cache is keyed by the *profile string as written on the respawn point*, which
  may name several configuration sections at once. Two points with the same profile share one
  parsed row list.

## `make_respawn_entity`

**Contract** — builds a prototype from a section name, an attachment flag set and an
ammunition count. Creates the entity through the class factory, marks it as having no
identifier, no parent, no phantom and no respawn of its own, and — when it is a weapon —
loads its magazine and applies the attachments.

```text
FUNCTION make_prototype(section, addons, ammo) -> entity
  entity = factory.create(section)
  FAIL WITH cannot_create IF none
  entity.id = entity.parent = entity.phantom = none
  entity.respawn_time = 0              # the item itself does not respawn; the point does
  IF entity is a weapon THEN
    entity.loaded = min(its magazine size, ammo)
    entity.attachments = addons
  RETURN entity
```

**Invariants** — the magazine is filled to capacity and then **clamped down** by the
configured count, so a configuration asking for more rounds than the weapon holds gets a full
magazine rather than an overfull one, and a configuration asking for zero gets an empty
weapon. Reading the clamp the other way round would let a configuration overfill.

Clearing the entity's own respawn time matters: the server has a general respawn mechanism
for entities, and a respawned item must not also respawn itself.

## `parse_string`

**Contract** — parses one configuration row: a comma-separated list whose first element is
the section and whose optional second, third and fourth are the delay in seconds, the
attachment flags and the ammunition count. Missing trailing elements default to zero. An
empty list fails.

**Invariants** — the delay is authored in seconds and stored in milliseconds; the conversion
happens here and nowhere else.

A delay of zero — which is what a row with only a section name gets — means the item
reappears on the first update after being taken. That is a usable behaviour, not a bug, but a
rebuild should be aware that "no delay configured" and "instant respawn" are the same value.

## `load_section_items` / `load_respawn_section`

**Contract** — reads the multiplayer respawn configuration file. A profile string may name
several sections; each section holds rows under indexed keys read until one is missing. Rows
that fail to parse are warned about and skipped; a missing section is an error and
contributes nothing. The assembled row list is cached under the profile string.

**Invariants** — the configuration file is re-opened and re-parsed **once per profile**, not
once per level, because the cache is checked before the load is attempted. With a handful of
profiles per level that is acceptable; the cost is a full parse of the file each time and a
rebuild should load it once.

**Notes** — if the cache insertion finds the key already present, the freshly built list is
discarded and the function reports failure rather than returning the existing entry. That can
only happen if two threads load the same profile at once, which does not occur; the branch is
defensive and its failure report is misleading.

## `add_new_rpoint`

**Contract** — registers a respawn point: resolves its profile to a row list (loading it if
needed), and for each row builds a prototype and places it at the point's position and
orientation. One point therefore yields several pickups — a point is a *loadout*, not a
single item.

**Invariants** — called only during level setup. The position and orientation are baked into
each prototype, so nothing needs to remember the point afterwards.

## `check_to_delete`

**Contract** — reports that an entity was destroyed. If it is a pickup's live instance, stamps
the take time so the countdown begins. If it is a loose level item, forgets it.

**Invariants** — a pickup is found by scanning for the identifier of its live instance, so the
identifier is the only link between a world object and the point it came from. The scan is
linear over every pickup on the level, run on every item destruction — acceptable at
multiplayer scale, and the obvious place for a reverse index in a rebuild.

## `update`

**Contract** — for each pickup that is counting down and whose delay has elapsed, respawns it
and clears the countdown.

## `respawn_all_items`

**Contract** — spawns every pickup immediately, ignoring the timers, and clears every
countdown. This is the round-start stock-up.

## `respawn_item`

**Contract** — clones a prototype into the world.

```text
FUNCTION respawn(prototype) -> entity_id
  packet.reset_for_writing()
  prototype.write_spawn(packet)          # the same serialization a level spawn uses
  packet.begin_reading()                 # discarding the message header
  spawned = server.process_spawn(packet, as the server's own client)
  RETURN spawned.id, or none
```

**Invariants** — going out through serialization and back in through the server's spawn
handler is the decision the whole class rests on: the respawned item takes the identical path
a level-authored item takes, including identifier allocation and the broadcast to clients.
Constructing the object directly would bypass all of it.

The packet is a single reused buffer, which is why respawning is not reentrant.

## `respawn_level_items` / `clear_level_items`

**Contract** — the second population. Destroys the loose items placed by the previous round
and replays a dedicated spawn file to place a fresh set.

```text
FUNCTION respawn_level_items()
  FOR EACH remembered level item
    entity = server.lookup(it)
    IF missing THEN CONTINUE              # already gone: a round can end mid-destruction
    IF it now has a parent THEN CONTINUE  # somebody is carrying it; leave it alone
    server.destroy(entity, reliably)
  forget them all

  IF the level has a round-respawn spawn file THEN
    FOR EACH chunk in it
      copy the chunk into a packet
      FAIL WITH not_a_spawn UNLESS the packet's message identifier is a spawn
      entity = server.process_spawn(packet, as client zero)
      IF entity THEN remember its identifier
```

**Invariants** — **an item a player is carrying is not destroyed**: the parent check is what
distinguishes a dropped item from an item in somebody's inventory, and destroying the latter
would tear an item out of a live inventory. Everything loose goes.

The spawn file is a chunked container whose every chunk is one complete spawn message —
the same format the level's own spawn file uses, so the server's spawn handler consumes it
unchanged. The file is optional; a level without one simply has no round-respawned items.

**Notes** — an entity that cannot be found is skipped rather than being an error, because a
round can end while a destruction is already in flight.

## `clear_respawns` / `clear_respawn_sections`

**Contract** — destroy every prototype and drop every pickup; and free every cached loadout
row list. Both run at teardown, and the first also runs at construction.
