# src/xrGame/alife_spawn_registry.cpp

> Owns the level's spawn file: loads it, verifies it matches the game graph and the save, and keeps the authored spawn records available for the whole session.

**Needs** — [`alife_spawn_registry.h`](alife_spawn_registry.h.md) · [`alife_spawn_registry_header.h`](alife_spawn_registry_header.h.md) · [`server_entity_wrapper.h`](server_entity_wrapper.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`xrAICore/Navigation/graph_abstract.h`](../xrAICore/Navigation/graph_abstract.h.md) · [`game_base.h`](game_base.h.md)
**Used by** — reached through its declarations in [`alife_spawn_registry.h`](alife_spawn_registry.h.md); callers name that, not this file.
**Tier floor** — T1: the spawn file is a frozen binary container held open and read in place

## Purpose

The **spawn file** is the authored contents of the world: one record per entity that the
game designers placed, across every level, in one file per game. This registry owns it.

Three things make it more than a file reader.

First, the records form a **graph, not a list**. A vertex is a spawn record; an edge with
a weight joins a record to a record it may produce. That is how the shipped data expresses
"this crate contains one of these three things, with these odds" and "this spawn point
produces a squad" — the randomization is authored into the graph's topology, not into a
script.

Second, the file carries three other things the engine needs at load: the artefact spawn
positions, the level's patrol paths, and (in the later format generations) the game graph
itself.

Third, it is the **identity** a save is validated against: a save records the spawn file's
identifier, and loading a save against a different spawn file is refused.

Spawn selection lives in
[`alife_spawn_registry_spawn.cpp`](alife_spawn_registry_spawn.cpp.md).

## State

```text
RECORD SpawnRegistry
  header                   : SpawnHeader        # version and the two identifiers
  spawns                   : Graph<SpawnRecord, weight: real, id: SpawnId>
  artefact_spawn_positions : list<LevelPoint>   # candidate positions, indexed by anomaly
  spawn_name               : text               # the spawn file's logical name
  spawn_roots              : list<SpawnId>      # vertices with no incoming edge
  spawn_story_ids          : map<SpawnStoryId, SpawnId>
  file                     : open reader        # the spawn file, held open for the session
  chunk                    : open reader        # the game graph's bytes within it
  game_graph               : GameGraph          # built from that chunk; owned here
  random                   : Random             # the registry's own stream
```

Invariants:

- the spawn file and the game-graph chunk stay **open for the session**, because the game
  graph and every spawn record are read in place from the mapped bytes rather than copied
  out. Closing either invalidates the whole world. That is the one genuinely T1 constraint
  in the alife layer;
- the registry **owns the game graph** and publishes it globally, so the graph's lifetime
  is the spawn file's lifetime;
- `spawn_roots` is derived at load and never changes;
- the registry has its own random stream, seeded from the cycle counter like the
  simulator's — so spawn selection is not reproducible across runs either.

## `load(spawn_name)` — starting a new game

**Contract** — resolves a spawn file by logical name under the spawn root, opens it, and
loads it. Fails hard if the file does not exist.

## `load(save_stream, game_name)` — loading a saved game

**Contract** — reads the spawn file's *name and identifier* out of the save, then opens
that spawn file and loads it with the save's identifier to verify against.

```text
FUNCTION load(save_stream, game_name)
  REQUIRE the save file exists
  REQUIRE save_stream has chunk SPAWN_CHUNK_DATA
  sub = open that chunk
  identity = open sub-chunk 0
    spawn_name = identity.read_string()
    saved_guid = identity.read_bytes(guid size)
  path = resolve spawn root + spawn_name + ".spawn"
  REQUIRE it exists  ELSE FAIL WITH "can't find spawn file"
  file = open(path)
  load(file, expect_guid = saved_guid)
```

**Invariants** — the save does not contain the world; it contains the *name of the world*
plus the differences from it. That is the decision this routine encodes, and it is why a
save is small and why it is worthless without the matching game data.

## `load(stream, expected_guid)` — the spawn file itself

**Contract** — the real loader. Reads five chunks in a fixed order, builds the game graph,
and validates three identities. Fails hard on any mismatch.

```text
FUNCTION load(stream, expected_guid)
  header.load(stream.chunk(0))
  REQUIRE expected_guid is absent OR equals header.guid OR the override flag is set
    ELSE FAIL WITH "saved game doesn't correspond to the spawn — delete saved game"

  spawns.load(stream.chunk(1))                    # the spawn graph
  artefact_spawn_positions.load(stream.chunk(2))
  patrol_path_storage.load(stream.chunk(3))       # FAIL WITH rebuild spawn if absent

  IF header.version >= PRIQUEL
    graph_chunk = stream.chunk(4)                 # the game graph travels inside the spawn file
  ELSE
    graph_chunk = open the standalone game graph file
  REQUIRE graph_chunk exists  ELSE FAIL WITH rebuild spawn

  game_graph = build from graph_chunk
  publish game_graph globally

  REQUIRE header.graph_guid == game_graph.guid OR the override flag is set
    ELSE FAIL WITH "spawn doesn't correspond to the graph — rebuild spawn"

  build_story_spawns()
  build_root_spawns()
```

**Invariants** —

- **Three identities are checked, in this order**: the save against the spawn file, the
  spawn file's format version against this build (inside the header load), and the spawn
  file against the game graph. Each has a different remedy and each says so — delete the
  save, rebuild the spawn, rebuild the spawn — because they are failures of different
  artefacts.
- A command-line flag suppresses the first and third. It exists for modders regenerating
  data against existing saves and it is genuinely unsafe; a rebuild should keep it and
  keep it opt-in.
- The game graph moved **into** the spawn file at one format generation. Before that it
  was a separate file alongside the game data. Both are supported because both shipped,
  and the version field is the discriminator. A rebuild targeting only one game may drop
  the older branch.
- The patrol-path chunk's absence is reported as a *version* mismatch rather than as a
  missing chunk, because the chunk was added in a later generation and a file without it
  is simply too old.

## `save`

**Contract** — writes the spawn registry's contribution to a saved game: the spawn file's
identity, and the per-record updates.

```text
FUNCTION save(stream)
  open chunk SPAWN_CHUNK_DATA
    open chunk 0
      stream.write_string(spawn_name)
      stream.write_bytes(header.guid)
    close
    open chunk 1
      save_updates(stream)
    close
  close
```

**Invariants** — only the *name and identifier* of the spawn file are saved, never its
contents. The world is reconstructed by reloading the same file; what the save adds is
each record's mutable half.

## `save_updates` / `load_updates`

**Contract** — one sub-chunk per spawn record, **keyed by the record's own vertex
identifier**, holding that record's update state.

```text
FUNCTION save_updates(stream)
  FOR EACH vertex IN spawns.vertices
    open chunk numbered vertex.id
      vertex.record.save_update(stream)
    close

FUNCTION load_updates(stream)
  FOR EACH sub-chunk (id, content) IN stream
    vertex = spawns.vertex(id)     # REQUIRE present
    vertex.record.load_update(content)
```

**Invariants** — this is the one place in the alife save format that is **self-describing
and order-independent**: the chunk identifier is the vertex identifier, so the reader
looks each record up rather than replaying an order. That makes the update section
tolerant of a spawn file whose vertex count changed — records that no longer exist are
simply absent. The rest of the save format has no such property, so the tolerance buys
nothing in practice, but it costs nothing either.

The vertex identifier must fit the spawn-identifier width, which is checked.

## `build_root_spawns`

**Contract** — computes the set of spawn records that are not produced by any other
record. These are the entry points of the spawn walk.

```text
FUNCTION build_root_spawns()
  all      = every vertex id
  produced = every edge's target vertex id, over every vertex
  sort and deduplicate both
  spawn_roots = all minus produced
```

**Invariants** — a root is a record nothing points at. Walking from the roots visits every
reachable record exactly as its authors intended, and never visits a "contents" record on
its own — the crate's contents are reached *through* the crate, with the crate's odds
applied. A rebuild that walked every vertex instead would spawn every possible outcome of
every container.

The set-difference approach means a **cycle** among spawn records would be unreachable
from any root and silently never spawn. Nothing in the shipped data has one, and the
format does not forbid it.

## `build_story_spawns`

**Contract** — indexes every spawn record that carries an authored *spawn story
identifier*, so a script can name a spawn point without knowing its vertex number.

```text
FUNCTION build_story_spawns()
  FOR EACH vertex IN spawns.vertices
    record = vertex.record
    IF record.spawn_story_id is the invalid value -> CONTINUE
    spawn_story_ids.insert(record.spawn_story_id, vertex.id)
```

**Notes** — duplicates are not detected: two records sharing a spawn story identifier
leave one of them unreachable by name, silently. The identifier values themselves come
from configuration and are validated for duplication there (see
[`alife_simulator_script.cpp`](alife_simulator_script.cpp.md)), but nothing checks that
they are used at most once in the spawn file.

## Destruction

**Contract** — destroys the game graph, then closes the graph chunk, then closes the
spawn file. In that order, because the graph reads from the chunk and the chunk is a view
into the file.
