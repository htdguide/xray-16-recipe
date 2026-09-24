# src/xrAICore/Navigation/PatrolPath/patrol_path_storage.cpp

> The level's patrol-path registry — built either from the level editor's authored chunk tree or from the pre-converted form the spawn compiler writes, and owning every path in it.

**Needs** — [`patrol_path_storage.h`](patrol_path_storage.h.md) · [`patrol_path.h`](patrol_path.h.md) · [`patrol_point.h`](patrol_point.h.md) · [`../../../Common/LevelGameDef.h`](../../../Common/LevelGameDef.h.md)
**Used by** — [`patrol_path_storage.h`](patrol_path_storage.h.md)
**Tier floor** — T2: chunked stream reads and an ordered insert.

## Purpose

Two formats carry the same paths, and this file reads both. The authored one comes from the
level editor and stores positions that still have to be snapped onto the navigation mesh; the
pre-converted one is written by the spawn compiler with the snapping already done. The engine
reads the second at runtime and the first only when it is itself acting as the compiler. Keeping
both readers in one place is what makes it obvious that they must produce identical registries.

## State

```text
RECORD PatrolPathStorage
  registry : map<text, PatrolPath>   # ordered by name; OWNS every path
```

**Invariants** — names are unique. The authored reader asserts it; the runtime reader asserts it
*and* logs it, because a duplicate there means the compiled level data is wrong rather than the
authored source. Neither recovers: the later path silently replaces nothing and the registry
keeps whichever the container's insert chose, so a duplicate is a content bug to be fixed
upstream.

Every path in the registry is owned and destroyed with it — except those added as aliases, which
are second names for a path the registry already owns and must not be destroyed twice.

## `load_raw`

**Contract** — reads the authored form. The stream's patrol chunk holds one sub-chunk per path;
each carries a format version, a name and the path's own body. A missing patrol chunk is not an
error — a level may have no patrol paths — but a malformed sub-chunk is: the version must be
present and must match exactly, and the name must be present. Blocks for the length of the read;
this is level-load work.

```text
FUNCTION load_raw(level_graph, cross_table, game_graph, stream)
  chunk <- stream.open(PATROL_PATHS)
  IF chunk is absent THEN RETURN          # a level with no patrol paths is legal
  FOR EACH sub_chunk IN chunk
    REQUIRE sub_chunk has a VERSION field AND it equals the one supported version
    REQUIRE sub_chunk has a NAME field
    name <- read the name
    REQUIRE name is not already in the registry
    registry[name] <- new PatrolPath(name).load_raw(level_graph, cross_table, game_graph, sub_chunk)
```

**Invariants** — the version is checked for *equality*, not for "at least". The engine reads one
version of this format and refuses anything else rather than guessing, which matches the policy
the rest of the engine applies to authored data.

## `load`

**Contract** — reads the pre-converted form: a count, then one sub-chunk per path, each holding
the name and the path's serialized graph as two further sub-chunks. Clears the registry first,
so a reload never accumulates. In debug builds each loaded path is stamped with its registry key
so diagnostics can name it.

**Notes** — the count and the entries live in two separate top-level chunks, and the entries are
addressed by ordinal. That means the reader must know the count before it starts and cannot
simply iterate — a shape inherited from the chunk container rather than chosen. A rebuild is
free to iterate the entries and ignore the stored count, but must still write it for the format
to round-trip.

## `save`

**Contract** — writes the pre-converted form: the count, then each entry as a numbered sub-chunk
containing the name and the graph. Ordinals are assigned in registry order, which is name order.

**Invariants** — the reader addresses entries by ordinal, so the writer must emit them densely
from zero.

## `add_alias_if_exist`

**Contract** — if a path with the first name exists, registers the same path under the second
name as well and reports it; otherwise reports nothing and changes nothing.

**Invariants** — an alias is a second key onto one path, not a copy. The registry therefore holds
entries it must not destroy, and destruction as written walks every entry — so aliasing a path
and then destroying the registry frees the same path twice. Every shipped call site adds aliases
to a registry that outlives the level and is never torn down with aliases present, which is why
this has never surfaced. A rebuild must make ownership explicit here: either the registry holds
shared references, or aliases live in a separate name-to-name table.

**Notes** — aliases exist because the same authored path is referenced under different names by
different games' scripts; the compatibility shim lives here rather than in every caller.
