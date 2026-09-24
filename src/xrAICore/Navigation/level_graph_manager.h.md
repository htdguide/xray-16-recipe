# src/xrAICore/Navigation/level_graph_manager.h

> Presents the level mesh's vertex array as one array of the current record shape, converting older file generations on load.

**Needs** — [`level_graph_space.h`](level_graph_space.h.md) · [`../../Common/LevelStructure.hpp`](../../Common/LevelStructure.hpp.md) · [Data: Level data](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`level_graph.cpp`](level_graph.cpp.md) · [`level_graph.h`](level_graph.h.md)
**Tier floor** — T1: it either points at a mapped file region or allocates and fills an array of frozen-layout records.

## Purpose

Five generations of the navigation mesh format shipped, and the engine loads all of them. Rather
than teach every query about every generation, this layer normalizes: the current generation is
used in place, straight out of the mapped file; every older one is converted, record by record,
into a freshly allocated array of current-generation records. Everything above this point sees
one shape.

## State

```text
RECORD MeshVertexArray
  vertices          : ref to LevelVertex array   # into the file, or into an owned allocation
  count             : int
  converted         : bool     # true when the array was allocated and must be released
```

**Invariants** — the array is released exactly when it was converted; a mapped-in-place array
belongs to the file reader. The array is one element longer than the vertex count when converted
— see below.

## Loading

**Contract** — dispatch on the file's version. The current generation is used in place with no
copy. Each older generation names the record shape it stored, and each such shape knows how to
widen itself into the current one. An unrecognized version is a fatal error: the engine refuses
the level rather than guessing.

```text
FUNCTION load(stream, vertex_count, version)
  SELECT version
    current        -> vertices <- stream.pointer(); converted <- false
    each older one -> vertices <- convert(stream, vertex_count, its record shape)
    otherwise      -> FAIL WITH "unsupported level graph version"

FUNCTION convert(stream, count, OldShape) -> array
  out <- allocate count + 1 records            # one spare; see Notes
  FOR i IN 0 .. count - 1
    out[i] <- widen(old_records[i])            # each old shape defines its own widening
  out[count] <- a recognizable marker
  RETURN out
```

**Notes** — the generations differ in how many bits a neighbour link gets and how wide a packed
position is, because levels grew: the oldest shape packs links in 23 bits and a position in five
bytes, the newest in 26 bits and six. Widening is per-field and lossless in that direction. The
cover data is the one asymmetric case: the oldest generation stored *one* set of four cover
values, and widening duplicates it into both the high and the low set, so a creature crouching on
an old level gets the same cover it would get standing. That is a deliberate approximation, not a
conversion bug, and a rebuild should reproduce it or old levels will play differently.

The array is allocated one record longer than needed and the spare is filled with a readable
marker. The stated reason is that some query reads one past the end; the guard makes that read
land in owned memory instead of faulting. A rebuild should find and fix the overrun rather than
reproducing the pad — but must keep the pad until it has.

## `begin` / `end`

**Contract** — the vertex range, however it was obtained. This is the only surface above this
layer, and it is what makes the format generations invisible to every query.
