# src/xrGame/ai/monsters/chimera/chimera_state_hunting_move_to_cover_inline.h

> The cover half of the unbuilt hunting behaviour: every contract point is a stub.

**Needs** — [`chimera_state_hunting_move_to_cover.h`](chimera_state_hunting_move_to_cover.h.md)
**Used by** — [`chimera_state_hunting_move_to_cover.h`](chimera_state_hunting_move_to_cover.h.md)
**Tier floor** — T3: stubs

## Purpose

Part of the unbuilt subtree described in [`chimera_state_hunting_inline.h`](chimera_state_hunting_inline.h.md).

## State

Stateless.

## Contract points

`initialize` delegates to the base and does nothing else. `execute` does nothing. `check_completion` always answers no, so the state would never end on its own. `check_start_conditions` is declared but not defined here — it is the one contract point a rebuilder would have to invent, and what it was meant to ask is the whole missing idea: *is there usable cover between me and my prey?*

**Notes** — The cover query the state would have needed exists in the engine: the navigation layer ships a per-vertex measure of how exposed each position is from each direction. A rebuild that wanted to finish this behaviour would ask that for a vertex near the prey's line of sight, not invent a new mechanism.
