# src/xrAICore/Navigation/game_graph_inline.h

> Loads the cross-level graph by pointing at the mapped file, walks its edges through stored byte offsets, and binds the right level's cross table when a level loads.

**Needs** — [`game_graph.h`](game_graph.h.md) · [`game_graph_space.h`](game_graph_space.h.md) · [`game_level_cross_table.h`](game_level_cross_table.h.md) · [`../../xrCore/FS.h`](../../xrCore/FS.h.md) · [Data: Level data](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`game_graph.h`](game_graph.h.md)
**Tier floor** — T1: the graph *is* the mapped file; every accessor is a pointer arithmetic step into it.

## Purpose

The behaviour of the game graph: how it is loaded, how a vertex's edges and death points are
found, how a level's cross table is located inside the same file, and how the graph is written
back out. The record layouts are in [`game_graph_space.h`](game_graph_space.h.md).

## State

```text
RECORD GameGraph
  reader              : mapped file region   # owned or borrowed; see below
  owns_reader         : bool
  header              : GameGraphHeader       # copied out, because it is variable-length
  vertices            : ref into reader       # the vertex array, read in place
  enabled             : list<bool>            # per-vertex blocking; the only mutable state
  cross_tables        : ref into reader       # start of the per-level cross-table blob
  current_cross_table : optional<CrossTable>  # the one for the level now loaded
  current_level_vertex: VertexId              # some vertex belonging to the current level
```

**Invariants** — everything except `enabled` and the bound cross table is a view into the file:
the graph owns no vertex or edge storage of its own. The file must therefore outlive the graph,
which is why ownership of the reader is tracked explicitly — the graph may be handed a stream it
did not open.

## Loading

**Contract** — read the variable-length header, check its version against the supported range,
point the vertex array at the bytes immediately following, mark every vertex accessible, and — for
files new enough to carry them — locate the per-level cross tables by stepping past the vertex,
edge and death-point arrays.

```text
FUNCTION load(stream, own)
  header <- read_header(stream)                 # consumes a variable number of bytes
  FAIL WITH "version mismatch" IF header.version outside supported range
  vertices <- current position in stream
  enabled  <- all true, header.vertex_count entries
  current_level_vertex <- invalid
  IF header.version >= PRIQUEL                  # the generation that embedded cross tables
    p <- vertices + header.vertex_count * size_of(GameVertex)
    p <- p + header.edge_count * size_of(Edge)
    p <- p + header.death_point_count * size_of(LevelPoint)
    cross_tables <- p
```

**Invariants** — the four arrays are laid out back to back in this order: vertices, edges, death
points, cross tables. A rebuild that parses rather than maps must still know that order, because
nothing else records where each begins.

**Notes** — the older file generation stores each level's cross table as a separate file in the
level's own directory; from the *Clear Sky* generation onward they are concatenated into the game
graph itself. Both are supported and the version decides which.

## `begin` / `value` / `edge_weight` — walking edges

**Contract** — a vertex's outgoing edges are a contiguous run in the edge array, found by adding
the vertex's stored byte offset to the base of the *vertex* array and taking the vertex's
neighbour count.

```text
FUNCTION begin(vertex_id, out_first, out_last)
  v <- vertex(vertex_id)
  out_first <- vertices_base + v.edge_offset       # a byte offset, not an index
  out_last  <- out_first + v.neighbour_count
```

`value` reads the destination identity from an edge; `edge_weight` reads its stored distance.
Neither consults the vertex, which is why the signature can be shared with graphs that need both.

**Notes** — the offset being relative to the vertex array rather than to the edge array is not a
detail: it is how the file avoids storing a separate base. A rebuild must apply the same base or
every edge lookup lands in the wrong place.

## `begin_spawn`

**Contract** — the same arrangement for a vertex's death points: offset from the vertex array
base, count from the vertex.

## `distance(from, to)`

**Contract** — the stored cost of the direct edge between two vertices. Scans the source's
adjacency for the destination. If the two are not adjacent this is a hard failure with a
diagnostic, not a returned sentinel — callers are expected to only ask about vertices they know
are neighbours, typically consecutive entries of a path the search just returned.

## `accessible` / `valid_vertex_id` / `set_invalid_vertex`

**Contract** — `accessible` reads and writes the per-vertex blocking flag. A vertex identity is
valid when it is below the header's vertex count; the invalid identity is all-ones in the 16-bit
identity width, which is guaranteed not to be valid because the count is stored in the same
width. `set_invalid_vertex` writes that value and asserts it does not validate.

## `mask`

**Contract** — match a four-byte terrain requirement against a vertex's four-byte type: each
position must be equal, unless the requirement's byte is 255, which matches anything.

## `set_current_level`

**Contract** — called when a level loads. Binds that level's cross table and remembers one game
vertex belonging to the level, which callers use as a cheap "somewhere on this level" handle.

```text
FUNCTION set_current_level(level_id)
  release the previously bound cross table
  IF header.version >= PRIQUEL
    p <- cross_tables
    FOR EACH level IN header.levels                # in stored order
      IF level.id != level_id
        p <- p + (the 32-bit length stored at p)   # skip this level's table
        CONTINUE
      current_cross_table <- CrossTable(view at p + 4, length at p)
      BREAK
  ELSE
    current_cross_table <- CrossTable(file "level.gct" in the current level's directory)
  current_level_vertex <- the first vertex whose level_id matches, else invalid
```

**Invariants** — the cross-table blob is a chain of length-prefixed blocks, one per level, in the
same order as the header's level table; there is no index, so finding the right one means walking
and skipping. Each block's first 32-bit word is its own total length, including that word.

The search for a vertex of the level is a linear scan from vertex zero and takes the first match.
It must find one: a level with no game vertices is a broken build, and the assertion says so.

## `save`

**Contract** — write the header, then the four arrays back out verbatim: vertices, edges, death
points, then each level's cross-table block, each block's length read from its own first word.
Used by the offline tools, not by the game.

**Notes** — the constructor that takes a file name also takes a version argument which it
explicitly ignores; the version actually enforced is the one in the file, checked against the
engine's supported range. A rebuild should drop the argument.
