# src/xrAICore/Navigation/game_level_cross_table_inline.h

> Attaches the mesh-to-game-vertex table either to a chunk of a file or to a region inside the game graph, and answers lookups by direct indexing.

**Needs** — [`game_level_cross_table.h`](game_level_cross_table.h.md) · [`../../xrCore/FS.h`](../../xrCore/FS.h.md)
**Used by** — [`game_level_cross_table.h`](game_level_cross_table.h.md)
**Tier floor** — T1: the table is read in place out of a mapped region.

## Purpose

Two ways in, one way out. The table exists in two shipped arrangements — a standalone file and a
block embedded in the game graph — and both resolve to the same thing: a header copied out and a
cell array pointed at.

## State

Stateless beyond what it attaches: the header is copied out and the cell array pointed at. The records are in [`game_level_cross_table.h`](game_level_cross_table.h.md).

## `CrossTable(file_name)` — the standalone form

**Contract** — open the named file, read the header from its first chunk, check the version,
then point the cell array at the payload of its second chunk. The chunk stays open for the
table's lifetime, because the cells are read out of it in place. A missing or malformed file is a
hard failure with a diagnostic, not a recoverable condition.

```text
chunk 0 : CrossTableHeader, as a memory image
chunk 1 : the cell array, one Cell per level-mesh vertex, in vertex order
```

## `CrossTable(buffer, size)` — the embedded form

**Contract** — the header is copied from the front of the region and the cell array is the
remainder. No chunk framing: the block inside the game graph is header-then-cells with nothing
between them. Version is checked the same way. Owns nothing — the game graph's mapped file is the
backing store, so the table must not outlive it.

**Notes** — the buffer length is accepted and unused. The cell count comes from the header, and
the caller is trusted to have handed over a region at least that large. A rebuild should validate
the length against the header instead; the game graph already knows both numbers.

## `vertex(level_vertex_id)`

**Contract** — return the cell for a mesh vertex by direct indexing. Bounds-checked against the
header's vertex count. Constant time, no allocation — this is the call the whole table exists to
make cheap.

## `header()` and the cell accessors

**Contract** — field reads: version, the two vertex counts, the two identity stamps; and from a
cell, the covering game vertex and the precomputed distance to it.

## Teardown

**Contract** — the standalone form closes its chunk and its file; the embedded form releases
nothing, because it owns nothing.
