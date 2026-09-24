# src/xrAICore/Navigation/ai_object_location_impl.h

> The half of an entity's navigation location that needs the graphs themselves — writing an identity with validation, and turning one back into a graph record.

**Needs** — [`ai_object_location.h`](ai_object_location.h.md) · [`game_graph.h`](game_graph.h.md) · [`level_graph.h`](level_graph.h.md)
**Used by** — [`ai_object_location.h`](ai_object_location.h.md) · [`ai_monster_utils.cpp`](../../xrGame/ai/monsters/ai_monster_utils.cpp.md)
**Tier floor** — T2: validated field access over two graphs.

## Purpose

Separated from [`ai_object_location_inline.h`](ai_object_location_inline.h.md) purely so that the
common case — an entity holding a location and reading its identities — does not drag the full
level mesh and game graph definitions into every entity's header. Only code that actually
dereferences a location into a graph record includes this.

The contracts are in [`ai_object_location.h`](ai_object_location.h.md).

## State

Stateless. The record is in [`ai_object_location.h`](ai_object_location.h.md).

## Exported units

- `game_vertex(record)` / `game_vertex(id)` — store the game identity, validating it against the
  loaded game graph; the record form derives the identity from the record first.
- `game_vertex()` — the game graph record for the stored identity; requires it to be valid.
- `level_vertex(record)` / `level_vertex(id)` / `level_vertex()` — the same three for the level
  mesh.

**Notes** — every one of these reaches the graphs through the process-wide AI space rather than
through a reference the location holds. That is the engine's service-locator cycle-breaker (see
[`SYSTEM-REQUIREMENTS.md` §7](../../../SYSTEM-REQUIREMENTS.md#7-build-order)) showing up at a very
small scale: a location is two integers and holding a graph reference per entity would double its
size. What is actually being reached for is "the graphs of the currently loaded level", and a
rebuild should pass that explicitly — as the patrol point does, which takes both graphs as
arguments for exactly this reason.
