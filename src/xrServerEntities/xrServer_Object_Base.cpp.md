# src/xrServerEntities/xrServer_Object_Base.cpp

> The common header every entity record carries — identity, placement, flags, custom data — and the framing that lets a record written by any engine version be read by this one.

**Needs** — [`xrServer_Objects.h`](xrServer_Objects.h.md) · [`xrMessages.h`](xrMessages.h.md) · [`clsid_game.h`](clsid_game.h.md) · [`object_factory.h`](object_factory.h.md) · [`alife_space.h`](alife_space.h.md) · [`script_value_container_impl.h`](script_value_container_impl.h.md) · [`gametype_chooser.h`](gametype_chooser.h.md) · [Data: level data — the spawn file](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence) · [Data: save games](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it defines an exact byte sequence that shipped files already contain.

## Purpose

This is the root of chapter 22. Every spawn record in every shipped level, every entity in
every save game and every entity creation on the wire begins with the bytes this file
writes, and is parsed by the reader this file implements. Two decisions dominate: the
**spawn record is a fixed prefix followed by a length-prefixed class-specific payload**, and
**the reader is version-gated while the writer is not** — the engine writes only the current
layout and reads every layout that ever shipped.

## State

### The record prefix — written by every entity, in this order

```text
RECORD SpawnPrefix                 # what Spawn_Write emits, in emission order
  message_tag       : int (16-bit)   # always the "spawn" message identifier
  section           : text           # zero-terminated: the configuration section, and the
                                     #   record's class-and-tuning identity
  name_replace      : text           # zero-terminated: the level designer's instance name,
                                     #   empty string when unnamed. Scripts address entities
                                     #   by this name, so it is content, not a comment.
  reserved          : int (8-bit)    # always written 0; was a game-mode byte before v121
  respawn_point     : int (8-bit)    # 0xFE means "use the coordinates in this record"
  position          : real x 3       # 32-bit floats
  angle             : real x 3       # 32-bit floats, radians
  respawn_time      : int (16-bit)   # seconds; multiplayer only
  id                : int (16-bit)   # this entity's identifier; 0xFFFF = unassigned
  id_parent         : int (16-bit)   # container or carrier; 0xFFFF = none
  id_phantom        : int (16-bit)   # 0xFFFF = none
  state_flags       : int (16-bit)   # see below; the version-present bit is forced on
  version           : int (16-bit)   # the writer's SPAWN_VERSION, present iff the bit above
  game_type         : int (16-bit)   # which multiplayer modes this record participates in
  script_version    : int (16-bit)   # from configuration, not from the engine
  client_data_size  : int (16-bit)   # length of the blob below
  client_data       : bytes          # opaque to the record; the client side's own state
  spawn_id          : int (16-bit)   # index of this record in the level's spawn file
  payload_size      : int (16-bit)   # bytes of the class payload that follows, INCLUDING
                                     #   these two bytes
  payload           : bytes          # whatever the concrete class writes
```

**Invariants**

- **`payload_size` is back-patched.** The writer reserves the two bytes, writes the payload,
  then seeks back and stores the distance from the reserved slot to the current position. A
  rebuild that cannot seek in its output buffer must size the payload first. The value is
  checked on both sides: a payload no larger than the size field itself means the record was
  never successfully written, and the engine refuses it by name — the one class exempt is the
  multiplayer spectator, which legitimately has an empty payload.
- **The identity is (section, class).** The class identifier is *not* in the record: it is
  looked up from the section's `class` key in configuration at construction time. So a spawn
  record is portable only against the configuration set it was authored with. This is why
  conformance criterion 7 is about loading a save *next to its game data*.
- **`0xFFFF` is the in-band "none"** for every identifier field. It is not a sentinel a
  rebuild may re-spell.
- **The script version comes from data, not code.** It is read once from configuration
  (`script` section, `current_server_entity_version` key) and cached for the process; absent
  section or key yields zero. It travels in the record so that a script-declared class can
  version its own payload independently of the engine's spawn version.

### State flags

```text
CONSTANT flag_object_local     = 1 << 0   # the receiver becomes the authority for it
CONSTANT flag_has_update       = 1 << 2   # an update record follows the spawn record
CONSTANT flag_as_player        = 1 << 3
CONSTANT flag_phantom          = 1 << 4
CONSTANT flag_version_present  = 1 << 5   # the version field follows the flags
CONSTANT flag_update_follows   = 1 << 6
CONSTANT flag_spawn_time       = 1 << 7
CONSTANT flag_denied           = 1 << 8   # do not create this entity
```

The writer sets the version-present bit unconditionally and then masks the local/as-player
bits according to whether the recipient is to own the entity. Reading a record whose version
parses as zero is the signal for "this stream is not a spawn record at all": the reader
rewinds two bytes and reports failure, which is how the level loader finds the end of the
spawn chunk.

### Fields the record no longer carries

```text
RECORD ObsoleteSpawnControl        # written by versions 83..111, skipped on read
  probability        : real
  spawn_flags        : int (32-bit)
  spawn_control      : text
  max_spawn_count    : int (32-bit)
  spawn_count        : int (32-bit)
  last_spawn_time    : int (64-bit)
  min_spawn_interval : int (64-bit)
  max_spawn_interval : int (64-bit)
```

A designer-facing repeat-spawning system that was removed at version 112. The reader must
still consume these bytes for older saves; the flag *names* survive in the code as a set of
enumerated bits nobody tests. A rebuild reads and discards.

## `Spawn_Write`

**Contract** — serializes the prefix above into a packet, then delegates to the concrete
class's state writer for the payload, then back-patches the payload length. Publishes the
script-visible field values into the script table first (see the value container), so that a
script that set a field on this entity sees it serialized. Never fails; asserts on an empty
payload.

**Invariants** — the local-ownership bit is set from the caller's argument, not from the
entity's own state, because the same record is sent to many recipients and only one of them
owns the entity.

## `Spawn_Read`

**Contract** — the inverse, and the only place version gating lives. Reads the fixed prefix,
then decides field-by-field what the writer's version would have emitted, then hands the
remaining payload and its declared length to the concrete class. Returns failure (having
rewound) when the version reads as zero.

```text
FUNCTION spawn_read(packet) -> bool
  REQUIRE packet.tag == spawn_message
  read section, name_replace, reserved_byte, respawn_point
  read position, angle, respawn_time
  read id, id_parent, id_phantom
  read state_flags
  IF state_flags HAS flag_version_present THEN read version
  IF version > 120 THEN read game_type ELSE game_type = defaults
  IF version == 0 THEN rewind 2 bytes; RETURN false     # not a spawn record
  IF version > 69 THEN read script_version
  IF version > 70 THEN
    size = (version > 93) ? read 16-bit : read 8-bit     # the blob outgrew one byte at 94
    read client_data[size]
  ELSE client_data = empty
  IF version > 79 THEN read spawn_id
  IF version < 112 THEN skip ObsoleteSpawnControl by version
  read payload_size
  REQUIRE payload_size > 2 OR record came from a text stream OR class is spectator
  delegate STATE_Read(packet, payload_size)
  RETURN true
```

**Notes** — the payload size is passed down but almost no class uses it; the payload is
self-describing given the version. It exists so that a reader *could* skip an unknown class,
and because the writer needed the assertion.

## `load` (from a save packet)

**Contract** — the save-restore path for the base record, distinct from spawn read: it
restores only the client-side opaque blob, because everything else in the prefix is
re-established by the spawn record the save also carries. The same width switch at version 93
applies.

## `spawn_ini`

**Contract** — materializes the per-entity configuration overlay. An entity's section may
name a `custom_data` file; its whole text is captured into the record at construction and
travels with it (it is a payload field of the alife object level, below), and this accessor
parses that text into a configuration object on first use and caches it. Parsing is deferred
because most entities never ask.

**Notes** — the text is captured, not the filename, so a save restores the overlay the entity
was created with even if the data files changed. When the alife simulation is running it
serves the file from its own cache instead of the filesystem, because thousands of entities
share a handful of overlay files.

## `CSE_Abstract` construction

**Contract** — builds a record from a section name: reads the class identifier from the
section, resolves the script-visible class number through the class registry, zeroes the
placement, sets every identifier to "none", captures the custom-data overlay text, and marks
the record as alife-controlled by default. Fails hard if the section has no `class` key — an
entity with no class is not a thing this engine can represent.

## `validate`

**Contract** — a per-class hook asked before a record is accepted, returning true here. Only
the hanging lamp overrides it (a lamp with no renderer generation selected is rejected).

## `match_configuration`

**Contract** — asks whether this record is meaningful in the current build and renderer.
Returns true here. It gates the online/offline decision: a record that does not match its
configuration may never come online and must be allowed to go offline. The lamp uses it to
keep renderer-specific lamps out of the wrong renderer; debug builds use it to hide
designer-only entities.

## The cast family

**Contract** — a set of downcast accessors, one per interesting base (inventory item, trader,
schedulable, creature, monster, human, zone, group, weapon, ammo, detector, PDA and a dozen
more), each returning nothing by default and being overridden to return itself by the class
that implements it.

**Notes** — this is the incidental half of a deliberate decision: the hierarchy is *wide
multiple inheritance without virtual bases*, so a language-level dynamic cast would have to
walk a lattice. The decision that survives is that **a record answers "am I a trader?" by
answering a question, not by having its type inspected**, and that the answer set is closed
and enumerated here. A rebuild with interfaces and interface queries writes this as the
interface list; a rebuild with sum types writes it as the variant's cases. See
[`smart_cast.h`](smart_cast.h.md) for the machinery that turns these answers into a cast.

## `CPureServerObject`

**Contract** — the four-way serialization vocabulary (read/write against a file, read/write
against a packet) with empty bodies, so that every record has all four entry points and
overrides only the ones it needs. A rebuild does not need the empty base; it needs the fact
that **four distinct serialization directions exist for one record** and that they are not
the same bytes: spawn, save, update-in and update-out.
