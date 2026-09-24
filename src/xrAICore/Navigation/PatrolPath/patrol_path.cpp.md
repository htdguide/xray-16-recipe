# src/xrAICore/Navigation/PatrolPath/patrol_path.cpp

> Reads one authored patrol path out of the level editor's layout — its waypoints and its probability-weighted links — into a small in-memory graph.

**Needs** — [`patrol_path.h`](patrol_path.h.md) · [`patrol_point.h`](patrol_point.h.md) · [`../graph_abstract.h`](../graph_abstract.h.md) · [`../../../Common/LevelGameDef.h`](../../../Common/LevelGameDef.h.md)
**Used by** — [`patrol_path.h`](patrol_path.h.md)
**Tier floor** — T2: a chunked stream read, field by field; nothing is aliased as a memory image.

## Purpose

A patrol path is authored in the level editor and shipped inside the level's way-point file.
This file is the read side of that format, and the decision it encodes is that the authored
form is *not* the runtime form: the editor stores positions, the runtime needs mesh vertices,
and the conversion happens here, once, at load.

## State

```text
RECORD PatrolPath
  vertices : map<int, PatrolPoint>   # keyed by the index the editor wrote them in
  edges    : per vertex, list<(target vertex, probability : real)>
  name     : text                    # debug builds only; the registry key is authoritative
```

**Invariants** — vertex identifiers are the positions in the authored point list, counting from
zero, and the link records refer to them by that number. Nothing renumbers them, so a rebuild
must preserve authored order.

## `load_raw`

**Contract** — reads one path from an open chunk. Two chunks, each mandatory — their absence is
a hard failure, not an empty path. First the waypoints, then the links. Each waypoint reads
itself and snaps itself onto the navigation mesh as it goes, which is why the three navigation
structures are threaded through. Returns the path so the caller can construct and load in one
expression.

```text
FUNCTION load_raw(level_graph, cross_table, game_graph, stream)
  REQUIRE stream has the POINTS chunk
  n <- read 16-bit count
  FOR i IN 0 .. n-1
    add_vertex(PatrolPoint.load_raw(level_graph, cross_table, game_graph, stream), index = i)

  REQUIRE stream has the LINKS chunk
  m <- read 16-bit count
  FOR 1 .. m
    from        <- read 16-bit vertex index
    to          <- read 16-bit vertex index
    probability <- read 32-bit real
    add_edge(from, to, probability)
```

**Invariants** — counts are 16-bit, so a path holds at most 65535 waypoints and 65535 links.
Authored paths are a handful of points; the width is not a constraint in practice but it is
part of the frozen format.

**Notes** — edges are directed and are added exactly as written. An authored path that should be
walked in both directions carries a link in each direction, and a rebuild must not helpfully
symmetrise them — a one-way patrol loop is a thing level designers author deliberately.

The probability on a link is not normalised here and need not sum to one across a waypoint's
outgoing links; whoever chooses among them normalises.

## `load` (debug builds)

**Contract** — reads the pre-converted runtime form through the inherited graph serialization,
then walks the waypoints re-establishing each one's back-reference to its owning path.

**Notes** — the back-reference exists only so that an assertion about a misplaced waypoint can
name the path it is in. It is compiled out of a shipping build and a rebuild may drop it — at
the cost of a diagnostic that is genuinely useful, because a waypoint that fails to land on the
mesh is a content error and the message is how a level designer finds it.

## Notes

The file defines a named constant holding one particular patrol path's name. Nothing reads it.
It is a leftover from debugging a specific level and carries no meaning; a rebuild should not
reproduce it.
