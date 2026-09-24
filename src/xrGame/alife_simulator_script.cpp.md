# src/xrGame/alife_simulator_script.cpp

> The script layer's window onto the whole alife simulation — and the only place that knows how to spawn an entity into a *live* parent rather than an offline one.

**Needs** — [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_story_registry.h`](alife_story_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_spawn_registry.h`](alife_spawn_registry.h.md) · [`alife_registry_container.h`](alife_registry_container.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`restriction_space.h`](../xrServerEntities/restriction_space.h.md) · [`xrServer.h`](xrServer.h.md) · [`Level.h`](Level.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a registration table, plus several adapters that carry real logic

## Purpose

Most `_script` files in this directory are pure registration tables. This one is not. It
exports the alife simulator under the name `alife_simulator`, publishes the free function
`alife()` that every shipped script calls to reach it, and generates two enumerations from
configuration. But it also contains a handful of adapters that exist because **the naive
binding would be wrong** — above all, the rule that spawning an item into a parent which is
currently online cannot go through the ordinary spawn path.

The exported names here are frozen by conformance criterion 10; they are the surface the
entire shipped script layer is written against.

## State

```text
RECORD ScriptExports
  story_ids       : list<(text, int)>   # built once, from configuration
  spawn_story_ids : list<(text, int)>   # ditto
```

Both are generated at registration time from the game's own configuration and cached for
the process, not the session — regenerating them on a script-engine restart is
unnecessary because the configuration cannot change under a running process.

## `alife` — the global accessor

**Contract** — a free script function yielding the current simulator, or nothing when
there is none (multiplayer, or before a game starts). This is the entry point for
essentially every shipped script; a rebuild must publish it under exactly this name.

## Object lookup

**Contract** — `object` is exported three ways, and the differences are deliberate:

- **by identifier** — rejects the invalid identifier with a script-log error rather than
  a crash, then looks up tolerantly (a missing object yields nothing).
- **by identifier with an explicit tolerance flag** — the raw registry lookup; a script
  passing "do not tolerate" gets a hard failure on a missing entity, which is how a
  script asserts an entity must exist.
- **by display name** — a **linear scan of the whole object registry** comparing display
  names. This is the debug-shaped one: it is O(entity count) per call and there is no
  index. It exists because the display name is what appears in logs and in the debugger,
  so a script author wants to look one up. A rebuild should keep it and say in the
  documentation that it is not for per-frame use.

`story_object` resolves an **authored story identifier** instead — a stable name a level
designer gives an entity so scripts can address it without knowing its runtime identifier.
Tolerant of absence, because a story entity may legitimately not exist yet.

## `create` — four spellings, one hard problem

**Contract** — all four create a server object and return it. They differ in where the
entity comes from and where it goes.

```text
create(spawn_id)
    # Instantiate the authored spawn record with this identifier.
    vertex = spawns.graph.vertex(spawn_id)     # FAIL WITH invalid spawn id if absent
    RETURN simulator.create(vertex.template, spawn_id)

create(section, position, level_vertex, game_vertex)
    # A free-standing item, no parent.
    RETURN simulator.spawn_item(section, position, level_vertex, game_vertex, no parent)

create(section, position, level_vertex, game_vertex, parent_id)
    RETURN spawn_into_parent(...)              # see below

create(section, position, level_vertex, game_vertex, parent_id, register?)
    IF register? -> spawn_into_parent(...)
    ELSE         -> build the entity but do NOT register it; hand it back to the script
```

The fourth form is the interesting addition: it returns an **unregistered** entity so the
script can edit its state before it enters the world, and then calls `register` (below) to
commit it. Without it, a script wanting a non-default entity had to spawn it and then
mutate a live object, which the online case makes unsafe.

### `spawn_into_parent` — why an online parent is different

```text
FUNCTION spawn_into_parent(section, position, level_vertex, game_vertex, parent_id)
  IF parent_id is the invalid identifier
    RETURN spawn_item(..., no parent)

  parent = objects.lookup(parent_id, tolerate_missing = true)
  IF parent is absent
    log an error; RETURN none

  IF parent is offline
    RETURN spawn_item(..., parent_id)          # the ordinary path is correct

  # The parent is ONLINE: the item must arrive as a spawn MESSAGE, not as a record.
  packet = new packet tagged SPAWN
  packet.write_string(section)
  draft = spawn_item(..., parent_id, register = false)   # build it, do not register
  draft.write_spawn(packet, as_save = false)
  server.release_identifier(draft.id)
  destroy draft
  RETURN server.process_spawn(packet, from = the server's own client identifier)
```

**Invariants** — this is the load-bearing decision of the whole file, and it is invisible
from the script side. An **offline** parent is a record; adding a child to it is a record
edit. An **online** parent is a live client object with an inventory the client side is
simulating; a record edit would not reach it, so the item must be introduced the same way
the network introduces one — as a spawn message the server processes, which creates both
the server record and the live client object and runs the attachment on both sides.

The draft entity is built purely to serialize it. Its identifier is returned to the
allocator and the draft destroyed before the message is processed, so the entity that
ends up in the world gets a fresh identifier allocated by the message path. A rebuild that
skips the draft and serializes the parameters directly reaches the same place with less
ceremony, but must produce byte-identical spawn content, because the message path parses
it with the ordinary reader.

The message is written and then immediately re-read (rewinding past its own tag) rather
than being sent anywhere. It is a local round trip through the network format — the
engine talking to itself in the only language the spawn path speaks.

### `create_ammo`

**Contract** — the same two-path structure, specialized for ammunition, with one extra
step: the round count is set on the entity before it is committed.

**Invariants** — the requested count must not exceed the section's declared box size, and
that is checked and fatal. A script asking for more rounds than a box holds is a script
error, not something to clamp, because clamping silently would make an economy exploit
look like it worked.

Note the asymmetry with `spawn_into_parent`: when the parent is absent or offline, the
ordinary spawn path is used **with registration**, and the count is set afterwards on the
already-registered entity. That is safe only because an offline entity's state is a
record nobody is reading. A rebuild should set the count before registration in both
paths.

## `register` — commit a draft entity

**Contract** — takes an entity built but not registered (the fourth `create` form) and
introduces it through the same spawn-message round trip. Consumes the draft: its
identifier is released and it is destroyed, and a **different** object comes back.

**Notes** — a script must use the returned object and discard the one it passed in. That
is a sharp edge in the surface and a rebuild with a safer object model should remove it,
but the shipped scripts are written against it.

## `clone_weapon`

**Contract** — creates a new weapon of a given section carrying the *state* of an existing
magazined weapon: its flags, its attachments, its condition, its ammunition type, its
applied upgrades and both round counts. Returns nothing if the source is not a magazined
weapon. Registers the clone unless asked not to.

**Invariants** — the copied field set is explicit and partial. This is not a general
clone: it is "give me the same gun as a different section", which is what a weapon-upgrade
or weapon-conversion script needs. Fields not listed — position, parent, identifier,
display name — come from the new entity. A rebuild adding a field to magazined weapons must
decide whether it belongs in this list, and nothing will remind it to.

## `release`

**Contract** — destroys an entity. Two paths, again split on online state.

```text
FUNCTION release(object)
  REQUIRE object exists and is an alife object
  IF object is offline
    simulator.release(object, destroy_on_server = true)
    RETURN
  # Online: send a destroy event through the level's message path instead.
  packet = EVENT(server_time, GE_DESTROY, object.id)
  level.send(packet, reliable and ordered)
```

**Invariants** — symmetric with the spawn split and for the same reason: a live object
must be destroyed through the event path so both sides of the simulation agree. The
original calls this path a hack; it is not — it is the only correct thing to do — but the
fact that the two paths look nothing alike is a real asymmetry a rebuild can improve by
routing both through the event system.

The boolean second parameter is accepted and ignored, preserved because shipped scripts
pass it.

## Switching, restrictions and death

**Contract** — thin adapters over simulator methods, each exported under a script name:

- `set_switch_online` / `set_switch_offline` — pin an entity's promotion eligibility.
- `set_interactive` — whether an entity participates in offline encounters.
- `switch_distance` / `set_switch_distance` — read and write the online/offline radius.
  Exported under both names because the setter was originally an overload of the getter
  and scripts exist that use either.
- `kill_entity` — three forms: with an explicit graph vertex and killer, with a vertex,
  and with neither (the creature dies where it stands).
- `add_in_restriction` / `add_out_restriction` / `remove_in_restriction` /
  `remove_out_restriction` / `remove_all_restrictions` — attach and detach movement
  restrictors by entity identifier. The in/out distinction is which way the volume
  constrains: a permitted region or a forbidden one.
- `teleport_object` — move an entity to a graph vertex, level vertex and position.

## Information portions

**Contract** —

- `has_info(entity, info_id)` — does this character know this authored information
  portion? A linear search of that character's known-information list.
- `dont_has_info` — the negation, exported separately. The original's own comment calls
  it absurd and explains it: script authors asked for it. A rebuild should export it, for
  the same reason.
- `iterate_info(entity, callback)` — calls back with every information portion the
  character knows.

**Notes** — the lookup is a linear scan of a per-character list on every call, and
`has_info` is among the most frequently called functions in the shipped script layer
(dialogue preconditions test it constantly). A rebuild should make the per-character
collection a set; nothing observable changes and the cost goes away.

## Iteration and bulk access

**Contract** —

- `iterate_objects(callback)` — calls back with every registered server object; the
  callback returning true stops the walk early.
- `get_children(object)` — the entity's containment list, as a script-iterable sequence.
- `actor` — the player's server record, from the graph registry.
- `level_id` / `level_name` — the current level's identifier, and any level's authored
  name by identifier.
- `spawn_id(spawn_story_id)` — resolve an authored spawn-story identifier to a spawn
  record identifier.
- `valid_object_id` — whether an identifier is the invalid sentinel.
- `set_objects_per_update` / `set_process_time` — tune the offline scheduler's per-pass
  budget and the update manager's time budget from script.
- `set_start_position` / `set_start_game_vertex_id` — free functions that override where
  a new game begins, consulted by the loading path.

## The two generated enumerations

**Contract** — `story_ids` and `spawn_story_ids` are published as script enumerations
whose members come from named sections of the game's configuration, so that a script can
write a story identifier by name. Generation validates each entry and fails hard on a
violation:

```text
FUNCTION generate_ids(section) -> list<(name, value)>
  REQUIRE the section exists
  FOR EACH line (key, _) IN section      # key is the numeric value, value is the name
    name = the line's value, whitespace-trimmed
    REQUIRE name contains no space        ELSE FAIL WITH invalid description
    REQUIRE name is not the reserved invalid name  ELSE FAIL WITH redefinition
    REQUIRE name not already present      ELSE FAIL WITH duplicate
    append (name, parse_int(key))
  append (reserved invalid name, invalid value)
```

**Invariants** — the configuration is inverted relative to how it reads: the **line's
key is the numeric identifier and its value is the name**. The identifiers are therefore
authored numbers and are frozen by the level data, not assigned here.

The three validations are all fatal because each produces a silently wrong script world:
a name with a space cannot be written as a script identifier at all; redefining the
reserved invalid name makes "no story identifier" collide with a real one; and a duplicate
name means two entities answer to the same script constant. The reserved invalid entry is
appended last so it is always present regardless of the configuration.

**Notes** — the enumerations are generated once per process and reused across script-engine
restarts, which is safe because their source cannot change. The original wraps each in an
empty placeholder type purely because the binding layer attaches enumerations to classes;
that is incidental, and a rebuild publishes two tables.
