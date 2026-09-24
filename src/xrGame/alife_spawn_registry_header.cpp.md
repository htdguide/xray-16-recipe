# src/xrGame/alife_spawn_registry_header.cpp

> The spawn file's first chunk: the format version and the two identifiers that tie a spawn file, a game graph and a saved game together.

**Needs** — [`alife_spawn_registry_header.h`](alife_spawn_registry_header.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`Common/LevelStructure.hpp`](../Common/LevelStructure.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: a fixed byte layout read from a frozen file

## Purpose

Six fields, read in order, and their order is the format. This chunk is what makes the
three-way identity check in
[`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md) possible.

## State

```text
RECORD SpawnHeader                # read in exactly this order
  version     : int (32-bit)      # the spawn-file format generation
  guid        : bytes (16)        # this spawn file's identity
  graph_guid  : bytes (16)        # the game graph this spawn file was built against
  count       : int (32-bit)      # how many spawn records
  level_count : int (32-bit)      # how many levels the game graph spans
```

**Frozen.** The field order, the widths and the identifier size are all fixed by the
shipped files.

## `load`

**Contract** — reads the six fields and verifies the version against this build's expected
generation. Fails hard on a mismatch, naming the file.

```text
FUNCTION load(stream)
  version = stream.read_int32()
  REQUIRE version matches this build's spawn-format version   ELSE FAIL WITH mismatch
  guid        = stream.read_bytes(16)
  graph_guid  = stream.read_bytes(16)
  count       = stream.read_int32()
  level_count = stream.read_int32()
```

**Invariants** —

- The version is checked **first**, before any other field is read. That is deliberate:
  every field after it is only guaranteed to be where it is if the version matches, so
  reading further before checking would interpret arbitrary bytes.
- The version check is an equality against a build-time constant and cannot be suppressed
  — unlike the two identifier checks, which the command-line override can bypass. A spawn
  file of the wrong generation is unreadable; a spawn file of the right generation built
  from different source data is merely inconsistent, and a modder may knowingly accept
  that.
- Two identifiers, not one, and they answer different questions. The file's own
  identifier answers *is this the same world the save was made in*; the graph identifier
  answers *was this spawn file built against the game graph we just loaded*. A spawn file
  and a game graph that disagree produce entities on vertices that do not exist, which is
  why the remedy is to rebuild the spawn rather than to delete the save.

`count` and `level_count` are read and stored but not used to validate anything — the
graph's own vertex count is what the loader reports. They are available for tools.

## Notes

There is no `save`. The spawn file is never written by the engine; it is produced by the
offline level compiler and read as shipped. What a saved game stores is the file's *name
and identifier*, written by the spawn registry, not this header.
