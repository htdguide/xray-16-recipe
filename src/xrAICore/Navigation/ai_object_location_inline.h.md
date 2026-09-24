# src/xrAICore/Navigation/ai_object_location_inline.h

> The part of an entity's navigation location that needs no graph definitions — initialization to invalid, and the two identity reads.

**Needs** — [`ai_object_location.h`](ai_object_location.h.md) · [`level_graph.h`](level_graph.h.md)
**Used by** — [`ai_object_location.h`](ai_object_location.h.md)
**Tier floor** — T2: field reads and two constants.

## Purpose

Holds the half of the location's behaviour that is reachable from almost anywhere: setting both
identities to invalid, and reading them back. The other half — everything that turns an identity
into a graph record — is in [`ai_object_location_impl.h`](ai_object_location_impl.h.md), because
it needs the full graph definitions and pulling those into every entity's header would be
expensive.

The contracts are in [`ai_object_location.h`](ai_object_location.h.md).

## State

Stateless. The record is in [`ai_object_location.h`](ai_object_location.h.md).

## Exported units

- construction and `init` / `reinit` — set both identities to their graphs' invalid values, or to
  an all-ones value of the right width when the graph in question is not loaded.
- `level_vertex_id()` / `game_vertex_id()` — read the stored identities, unvalidated.

**Notes** — the split between this file and the implementation file is an include-cost decision,
not a design one. A rebuild whose compilation model does not punish transitive dependencies
should merge the three files into one.
