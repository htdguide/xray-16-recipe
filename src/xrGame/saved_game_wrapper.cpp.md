# src/xrGame/saved_game_wrapper.cpp

> Peeks into a save file for four facts — clock, level, level name, actor health — by decompressing it and reading exactly the actor's record and the game graph, then throwing the rest away.

**Needs** — [`saved_game_wrapper.h`](saved_game_wrapper.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_simulator_header.h`](alife_simulator_header.h.md) · [`alife_spawn_registry.h`](alife_spawn_registry.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: reads a frozen on-disk layout, decompressing a whole-file image into one allocation

## Purpose

The load menu needs to describe each save without restoring it. That sounds like a header
read and is not: the save format puts no summary at the front, so the level and the health
have to be dug out of the world snapshot itself. This file is the excavation, and it is worth
reading as a statement about the format — **the save has no metadata block, and adding one
is the single cheapest improvement a rebuild can make here.**

## State

`Stateless` as a module; the four recovered facts live in the record declared in
[`saved_game_wrapper.h`](saved_game_wrapper.h.md).

## The save file's outer layout

```text
save file (uncompressed prefix, then a compressed body)
  marker         : int (32-bit)   # all-ones. Distinguishes a save from anything else
  version        : int (32-bit)   # refused if older than this build's alife version
  source_count   : int (32-bit)   # the decompressed size of the body
  body           : bytes          # compressed; the remainder of the file
```

**Invariants** — the three leading fields are exactly three machine words, and the
decompressor is handed *the file length minus three words* as the compressed extent. That
arithmetic is the format: there is no compressed-length field, and a rebuild that adds or
removes a header field must adjust it.

## `saved_game_full_name`

**Contract** — resolves a bare save name plus an extension against the saves logical root and
answers the full path. The only place the saves root is named.

## `saved_game_exist`

**Contract** — true if a save by this name exists under the current extension **or** under
the legacy one. Two extensions are supported because the fork changed the extension and
existing installations have saves under the old one; the current extension is tried first
everywhere, so a save present under both resolves to the current.

## `valid_saved_game`

**Contract** — two forms. Over a stream: false unless the file is at least two words long,
the leading marker is the all-ones value, and the version is at least this build's alife
version. Over a name: resolves the name under either extension, opens it, applies the stream
form, and closes. A missing file is invalid, not an error.

**Invariants** — the version comparison is **at least**, not equal. A save from a *newer*
build passes this check and then fails somewhere deeper. The engine's stated policy is to
refuse mismatched versions rather than guess; this check implements only half of it.

**Notes** — the minimum-length test admits a file of exactly two words, which is one word
short of the three the constructor then reads. Unreachable in practice because a real save
always carries a body, but a rebuild should test against the full header.

## Construction — the peek

**Contract** — opens the named save (current extension, else legacy; asserts it exists),
validates it, decompresses the body whole into one allocation, and reads four facts out of
it. Blocks for the decompression, which is the whole save. Allocates the full decompressed
size, which for a late-game save is tens of megabytes — **this runs once per entry on the
load screen**, and that is a real cost a rebuild should notice.

Every failure after the validity check leaves the clock and health already recovered and sets
the level identifier to all-ones with an empty name. Failure is never signalled.

```text
FUNCTION construct(save_name)
  path = resolve(save_name)                      # current extension, else legacy
  REQUIRE path exists

  IF NOT valid_saved_game(path) THEN
    # An unreadable or outdated save still gets a plausible clock: the campaign's
    # configured start time, full health, no level.
    game_time = configured_alife_start_time
    actor_health = 1.0; level_id = none; level_name = ""
    RETURN

  body = decompress(file, declared_size)         # the whole snapshot, in one buffer

  game_time = read_alife_clock(body)             # a time manager loaded purely to be asked
                                                 #   its clock and then discarded

  seek body TO the object chunk                  # required; absent means a corrupt save
  entity_count = read int                        # read and ignored
  first = deserialize_one_server_object(body)    # ALWAYS the actor: see invariants
  actor_health = first.health

  spawn_name = read_spawn_file_name(body)        # from the save's spawn sub-chunk
  IF absent OR that spawn file does not exist THEN level_id = none; RETURN

  # The graph tells which level a graph vertex belongs to. Reuse the running
  # simulation's already-open spawn file when it is the same one, rather than
  # re-opening it.
  spawn = (running alife has this spawn open) ? that file : open(spawn file)
  graph = spawn.graph_chunk OR the standalone graph file      # see notes
  IF neither exists THEN level_id = none; RETURN

  level_id = graph.vertex(first.graph_vertex).level_id
  level_name = graph.level(level_id).name IF that level exists
               ELSE localized("error")

  release graph, spawn (unless borrowed), first, body
```

**Invariants**

- **The actor is the first entity in the object chunk and carries identifier zero.** The peek
  depends on it absolutely: it deserializes exactly one record and asserts both facts. That
  the actor is entity zero is a property of how the world is spawned, asserted here and
  nowhere else, and a rebuild that assigns identifiers differently breaks this file before it
  breaks anything visible.
- The level is derived, not stored: the actor's *game graph vertex* is stored, and the graph
  says which level a vertex belongs to. So the peek cannot answer the level without a graph,
  which is why it goes to the trouble of finding one.
- A level identifier that the graph does not know resolves to the **localized error string**
  rather than to an empty name, so the load screen shows something is wrong instead of a
  blank row. A save whose *spawn file* is missing entirely resolves to an empty name instead.
  Two different unreadable states presented two different ways; a rebuild should pick one.
- The borrowed-spawn-file path must not close what it borrowed. The flag tracking that is the
  only resource decision in the file and it is easy to get wrong in a rebuild.

**Notes**

- The graph is looked for in the spawn file first and in a standalone graph file second. The
  second path exists **because the first game shipped the graph separately** and the later
  two embedded it in the spawn; supporting both is what lets one build read all three games'
  data. This is a concrete instance of the four-generations-of-data problem the system
  requirements describe.
- The time manager is constructed, loaded and destroyed solely to extract one number. The
  clock's deserialization is not separable from the manager in the original; a rebuild should
  expose the clock as a standalone field read.
- When the spawn file is opened rather than borrowed, the path variable still holds the
  *save* path from earlier — the spawn name is resolved into it by the existence test that
  precedes the open, so the open reads the spawn. This is a trap for a rebuild that passes
  the path explicitly, which it should.
