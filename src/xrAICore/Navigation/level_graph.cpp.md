# src/xrAICore/Navigation/level_graph.cpp

> Loads the level mesh and answers the question every AI query starts with — which mesh vertex is this world position standing on?

**Needs** — [`level_graph.h`](level_graph.h.md) · [`level_graph_manager.h`](level_graph_manager.h.md) · [`../../xrCore/FS.h`](../../xrCore/FS.h.md) · [Seam: Profiler](../../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging) · [Data: Level data](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`level_graph.h`](level_graph.h.md)
**Tier floor** — T1: the mesh is used as a mapped image and the vertex lookup is on the hot path of every entity's movement.

## Purpose

Two jobs. First, bring the mesh into memory: open `level.ai`, check its version, normalize its
vertex array, and derive the grid dimensions the coordinate system needs. Second — and this is
where the engine spends real time — answer "which vertex am I on" for a moving entity, dozens of
times per frame, without searching.

## State

```text
RECORD LevelGraph
  reader        : mapped file region     # owns level.ai for the level's lifetime
  header        : ref LevelMeshHeader     # read in place at the front of the file
  vertices      : MeshVertexArray         # in place, or converted from an older generation
  access_mask   : list<bool>              # one per vertex; the restrictor's fence
  level_id      : LevelId                 # which level this mesh is, once bound
  row_length    : int                     # cells along z; the packing modulus
  column_length : int                     # cells along x
  max_x, max_z  : int                     # the grid index of the bounding box's far corner
```

**Invariants** — `row_length` is the modulus the packed cell index is built on, so it must be
derived identically at load and at every pack and unpack; getting it wrong shifts every position
in the level by a row. Both dimensions are computed from the bounding box and the cell size with
a half-cell bias and a small epsilon, and then one is added: the grid must have a cell at the far
edge, not stop short of it.

```text
row_length    <- floor((box.max.z - box.min.z) / cell_size + epsilon + 1.5)
column_length <- floor((box.max.x - box.min.x) / cell_size + epsilon + 1.5)
```

The `1.5` is a half-cell rounding bias plus the inclusive extra cell, folded into one constant.

## Loading

**Contract** — open the mesh file for the current level (or a named directory, for the offline
tools), point the header at its first bytes, check the version against the supported range, hand
the rest to the vertex-array normalizer, derive the grid, and mark every vertex accessible. A
missing file is a hard failure that names the level and says to compile AI for it.

## `vertex_id(position)` — the indexed lookup

**Contract** — find the mesh vertex at a world position by its horizontal cell, choosing among
the vertices stacked in that cell by height. Returns the invalid identity when no vertex occupies
that cell. Requires the position to be inside the mesh's bounds; violating that is a programming
error.

```text
FUNCTION vertex_id(position) -> VertexId
  target <- pack_xz(position)
  first  <- binary search for the first vertex with cell index >= target
  IF none, or its cell index != target
    RETURN invalid
  best <- first
  best_y <- plane_y(best, position.x, position.z)
  FOR EACH v IN the run of vertices sharing that cell index, after the first
    y <- plane_y(v, position.x, position.z)
    best, best_y <- prefer(best, best_y, v, y, position.y)
  RETURN best

STEP prefer(best, y, candidate, cy, py)     # which stacked floor the position is on
  IF y <= py                                # current best is at or below the position
    IF cy <= py AND py - cy < py - y        # candidate is also below, and closer
      -> candidate
    ELSE -> best                            # a floor above never displaces one below
  ELSE                                      # current best is above the position
    IF cy <= py           -> candidate      # any floor below wins over one above
    IF cy - py < y - py   -> candidate      # else the nearer of the two above
    ELSE                  -> best
```

**Notes** — the preference rule is the whole content of this function and it encodes one idea: a
creature is standing *on* a floor, not under a ceiling, so a mesh vertex below the position always
beats one above it, however close the one above is. Only when every candidate is above does
nearness decide. A rebuild that simply picks the nearest height puts creatures on the wrong floor
of every multi-storey building in the game.

The comparison uses each candidate's *plane height at the query's horizontal position*, not the
vertex's own stored height — the cells are tilted, so the two differ by up to half a cell's slope.

## `vertex(position)` — the exhaustive lookup

**Contract** — scan every vertex in the mesh and return the one whose contour is nearest the
position. Linear in the mesh size, so hundreds of thousands of contour distance computations.
Used only as the last resort, when the caller has no current vertex to start from at all.

## `vertex(current, position)` — the incremental lookup

**Contract** — the call every moving entity makes. Given the vertex the entity was on last frame
and its new position, return the vertex it is on now. Always returns a valid vertex when one can
be found; falls back through progressively more expensive strategies. Timed, because it is on the
hot path.

```text
FUNCTION vertex(current, position) -> VertexId
  IF position is inside the mesh bounds
    IF current is valid AND position is inside current's cell
      RETURN current                        # the overwhelmingly common case: nothing moved far
    candidate <- vertex_id(position)        # binary search by cell
    IF candidate is valid
      IF current is invalid
        RETURN candidate
      IF candidate is one of current's four links
        RETURN candidate                    # adjacent: trust it unconditionally
      IF current is one of candidate's four links
        RETURN candidate                    # adjacent the other way: same
      IF plausible_height(current, candidate, position)
        RETURN candidate
  IF current is invalid
    RETURN vertex(position)                 # exhaustive; no starting point at all
  guessed <- guess_vertex_id(current, position)
  IF guessed != current
    RETURN guessed
  RETURN nearest of current and its four links, by contour distance

STEP plausible_height(current, candidate, position)
  y0 <- plane_y(current,   position.x, position.z)
  y1 <- plane_y(candidate, position.x, position.z)
  IF position.y <= y0     RETURN true       # we are below the old floor: accept the new vertex
  # we are above the old floor: reject the candidate only if it is more than a metre
  # further below us than the old one, in either direction
  RETURN abs((position.y - y1) - (position.y - y0)) <= 1
```

**Notes** — the height plausibility test exists to stop an entity from teleporting between
stacked floors. The binary search finds the best vertex *in a cell*, but the cell may contain the
floor above and the floor below; if the entity was demonstrably standing on one of them, a
candidate a metre or more further down is rejected as the wrong storey. The one-metre threshold
is the file's most arbitrary constant: it is smaller than a storey and larger than any single
step, and the source gives no derivation.

Adjacency short-circuits the height test entirely. If the two vertices are linked, the entity
could have walked from one to the other, so no amount of height difference is suspicious — which
is how the engine handles stairs.

## `guess_vertex_id` — the local sweep

**Contract** — search a fixed neighbourhood of grid cells around the position for the vertex whose
contour is nearest, and return it if it beats the current vertex. Returns the current vertex
unchanged when nothing better is found. Used when the indexed lookup failed — the position's own
cell holds no vertex — which happens when an entity walks off the mesh or the mesh has a hole.

```text
FUNCTION guess_vertex_id(current, position) -> VertexId
  anchor <- pack_xz(position) if the position is in bounds, else current's own cell
  x, z   <- unpack anchor
  best_distance <- distance from position to current's contour
  best          <- current
  FOR i IN [x - 4 .. x + 4] CLAMPED to the grid
    FOR j IN [z - 4 .. z + 4] CLAMPED to the grid
      cell <- i * row_length + j
      first <- binary search for the first vertex in that cell; SKIP if none
      pick the vertex in that cell whose contour is nearest the position
      IF abs(that contour point's height - position.y) >= 3   CONTINUE
      IF its distance >= best_distance                        CONTINUE
      best, best_distance <- it
  RETURN best
```

**Notes** — two constants. The sweep is four cells in each direction, giving a nine-by-nine
neighbourhood — wide enough to cross a small hole in the mesh, narrow enough that eighty-one
binary searches stay affordable. The three-metre vertical gate rejects candidates on another
storey, and is deliberately looser than the one-metre gate above because here there is no
adjacency evidence to fall back on. Neither number is derived in the source.

The sweep is over *grid cells*, each requiring its own binary search, because the vertex array is
sorted by packed cell index and there is no two-dimensional index over it. A rebuild with a
spatial index can answer this in one query, and should.
