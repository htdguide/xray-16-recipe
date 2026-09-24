# src/xrGame/alife_storage_manager.cpp

> Writes and reads a saved game: the chunk order, the compression wrapper, and the multi-pass restore that makes thousands of entities come back consistent.

**Needs** — [`alife_storage_manager.h`](alife_storage_manager.h.md) · [`alife_simulator_header.h`](alife_simulator_header.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`alife_spawn_registry.h`](alife_spawn_registry.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_group_registry.h`](alife_group_registry.h.md) · [`alife_registry_container.h`](alife_registry_container.h.md) · [`saved_game_wrapper.h`](saved_game_wrapper.h.md) · [`autosave_manager.h`](autosave_manager.h.md) · [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: stream ordering and a whole-buffer compression round trip

## Purpose

The save format is described in §5 as "a compressed chunked snapshot of every entity's
state plus the script layer's own serialized tables". This file is that sentence's
implementation, and its two load-bearing contents are **the order the sections are
written in** and **the four-pass restore**, neither of which is recorded anywhere in the
file.

## State

```text
RECORD StorageManager
  save_name : text    # the current save's file name, with extension; persists between saves
  section   : text    # the configuration section the simulation is rebuilt from
```

The save name is sticky: saving without naming a file reuses the last name, which is how
the quick-save and autosave paths work.

## The file layout

```text
RECORD SaveFile
  marker          : int (32-bit)   # all bits set; distinguishes this from the oldest format
  version         : int (32-bit)   # the alife format version
  uncompressed_size : int (32-bit)
  payload         : bytes          # compressed; the section stream below
```

```text
RECORD SectionStream                # the payload, uncompressed. Order IS the format.
  1. simulator header    # the version stamp, again, inside
  2. time manager        # the game clock
  3. spawn registry      # the spawn file's name and identity, plus per-record updates
  4. object registry     # every entity, as a parent-first tree of spawn+update packets
  5. registry container  # the nine persistent per-character stores
```

**Invariants** — the section order is fixed and unrecorded. Reordering it invalidates
every save and the failure is silent rather than diagnosed, because the only integrity
check is the version at the top.

The extension differs by which game is being run — the two older titles use one extension
and the newest another — so the same engine writes files the original of each game will
recognise.

## `save(name, update_name)`

**Contract** — serializes the whole world to a save file. Blocking; allocates a buffer the
size of the uncompressed snapshot plus a compressed one. Calls into the script layer twice.

```text
FUNCTION save(name, update_name)
  truncate name so that name + extension + saves path fits the path limit
  record it as the last saved game
  previous_name = save_name
  IF name is non-empty -> save_name = name + extension
  ELSE IF save_name is empty -> log "no file name specified"; RETURN

  call script hook "before_save"(save_name)      # a mod may record its own state now

  # 1. Serialize into memory, in section order.
  buffer = new memory stream
  header.save(buffer); time.save(buffer); spawns.save(buffer)
  objects.save(buffer); registry_container.save(buffer)
  uncompressed_size = buffer.length
  compressed = compress(buffer)

  # 2. Write the file.
  file = open(saves path + save_name)
  file.write_int32(all bits set)         # format marker
  file.write_int32(ALIFE_VERSION)
  file.write_int32(uncompressed_size)
  file.write_bytes(compressed)
  close

  call script hook "save"(save_name)      # a mod writes its sidecar now

  IF NOT update_name -> save_name = previous_name
```

**Invariants** —

- The whole snapshot is built **in memory first**, then compressed as one block, then
  written. Not streamed. A level's world is on the order of a few megabytes, so this is
  affordable; a rebuild streaming instead must still produce one compressed block, because
  the reader decompresses the whole payload before parsing it.
- The uncompressed size is written before the payload, because the decompressor needs the
  output size up front.
- The two script hooks bracket the file write, not the serialization, and both receive the
  final file name. That is what lets a mod write a parallel sidecar file next to the save
  and find it again on load. A rebuild must fire them at the same points and pass the same
  name, or every mod's custom save state desynchronizes from the engine's.
- `update_name` false means "write this file but do not make it the current save" — the
  autosave path, which must not change what a subsequent quick-save overwrites.
- The name is truncated to fit the path limit, silently. A rebuild should reject an
  over-long name instead; a truncated name collides with another save.

## `load(name)`

**Contract** — replaces the running world with a saved one. Fails soft (returns false,
restores the previous save name) if the file cannot be opened or is not a valid save;
fails hard on a structural problem after that point. Blocking, and the longest operation
in a session after startup.

```text
FUNCTION load(name) -> bool
  truncate name; resolve to a path under the saves root
  previous_name = save_name
  IF name is empty AND save_name is empty -> log; RETURN false
  IF name is non-empty -> save_name = name + extension

  stream = open(path)
  IF absent -> log; restore previous_name; RETURN false
  IF NOT valid_saved_game(stream) -> log mismatch; restore previous_name; RETURN false

  show the localized loading title naming the save
  unload()                     # destroy the current world entirely
  reload(section)              # rebuild empty registries

  uncompressed_size = stream.read_int32()
  payload = decompress(stream.remaining_bytes, uncompressed_size)
  close stream
  load_sections(payload, path)

  groups.on_after_game_load()  # squads re-resolve their memberships
  REQUIRE a player exists
  RETURN true
```

**Invariants** —

- The save is **validated before the world is destroyed**. That is why a bad save returns
  the player to the menu rather than to nothing. A rebuild that tears down first will turn
  every corrupt save into a crash.
- `unload` then `reload` is a full teardown and rebuild, not a reset: the registries are
  destroyed and recreated. Anything holding a reference into the old world across this
  point is dangling, which is why the load path runs with the level in a quiesced state.
- The remaining-bytes computation for the decompressor subtracts three 32-bit words —
  the marker, the version and the size — from the stream length. That arithmetic is the
  header layout restated; a rebuild reading the header field by field derives it instead.
- The crash-report file is pointed at the save being loaded, so a crash during or after a
  load names the save that produced it.
- The player must exist when the load finishes. A save without one is structurally
  impossible and the check is the last line of defence.

## `load_sections` — the four-pass restore

**Contract** — parses the decompressed section stream and brings every entity back. This is
the most order-sensitive routine in the save system.

```text
FUNCTION load_sections(buffer, file_name)
  call script hook "load"(file_name)      # a mod restores its sidecar before anything exists

  header.load(buffer)                     # version check first
  time.load(buffer)
  spawns.load(buffer, file_name)          # re-opens the spawn file, verifies identity
  graph.on_load()                         # the graph registry prepares for repopulation
  objects.load(buffer)                    # PASS 1: every entity is constructed and in the registry

  suppress entity registration hooks
  FOR EACH object IN objects              # PASS 2: registration, hooks deferred
    reserve object.id from the server's identifier allocator
    REQUIRE the allocator returned the same id
    register_object(object, add_to_object_registry = false)

  registry_container.load(buffer)         # PASS 3: the persistent per-character stores
  allow entity registration hooks

  FOR EACH object IN objects              # PASS 4: now run every entity's own hook
    object.on_register()

  IF a level is loaded
    level.autosave_manager.on_game_loaded()
    call script hook "after_load"(file_name)
```

**Invariants** — each pass exists because the one before it is not enough:

1. **Construct everything first.** Entities reference each other by identifier —
   parents, children, squad members, smart-terrain assignments — and no ordering of
   construction makes every reference resolvable at construction time.
2. **Register with hooks suppressed.** Registration puts each entity into the six
   registries so that lookups by identifier start working. The entity's *own* hook is
   deferred because it would reach for a world that is still only half-registered.
   Registration also does **not** re-add to the object registry — the entities are already
   there from pass 1, and adding again would trip the duplicate check.
3. **Load the persistent stores** only after every entity exists, because those stores are
   keyed by entity identifier.
4. **Run every entity's hook** with the world complete.

The identifier re-reservation in pass 2 is easy to miss and essential: the server's
allocator starts empty after the teardown, and every restored entity must reclaim the
exact identifier it had. The check that the allocator returned the same value is what
catches a save whose identifiers collide.

The script hooks bracket the whole restore — one before anything is read, one after
everything is registered — mirroring the save's pair.

## `save(packet)`

**Contract** — the network entry point: a save request arrives as a message carrying the
save's name and the update-name flag. Before serializing, it forces the client side to
flush its state into the server records.

```text
FUNCTION save(packet)
  level.client_send()     # push pending client state to the server
  level.client_save()     # have every live client object write itself into its server record
  name = packet.read_string()
  update_name = packet.read_byte() as bool
  save(name, update_name)
```

**Invariants** — the flush is the whole reason this wrapper exists. A live client object
holds state the server record does not have until it is asked for it; saving without the
flush writes a world one update behind the one the player sees. This is the single most
visible correctness requirement of the save path, and it is the reason saving is a
*request* rather than a direct call.

A rebuild that keeps the server record authoritative at all times — which the naming
suggests but the implementation does not achieve — deletes this step. The engine does not,
and conformance criterion 7 is tested against the result.
