# src/xrGame/ai/monsters/bloodsucker/bloodsucker_vampire.h

> Declares the vampire behaviour: the four-node tree that carries a bloodsucker from wanting a feed to being gone again.

**Needs** — [`state.h`](../state.h.md) · [`bloodsucker_vampire_inline.h`](bloodsucker_vampire_inline.h.md)
**Used by** — [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md) · [`bloodsucker_vampire_inline.h`](bloodsucker_vampire_inline.h.md)
**Tier floor** — T3: a declaration over the shared state contract

## Purpose

Declares the surface implemented in [`bloodsucker_vampire_inline.h`](bloodsucker_vampire_inline.h.md), which carries the whole substance: the node set, the cooldown shared across every bloodsucker in the world, and the partial-visibility rule that holds for the length of the tree.

The declaration pulls in the class-identifier table, because the state's legality test names the one entity class it is allowed to feed on.

## `BloodsuckerVampireState`

A composite state holding one field — the victim captured when the tree was entered — and overriding entry, substate selection, both exits, the start and completion tests, reference cleanup, the parameter fill and the forced-restart hook.

**Notes** — the captured victim is the reason this state overrides reference cleanup at all: an entity can be destroyed while the tree is mid-feed, and a behaviour holding a stale reference to it is the classic crash of this codebase. The rule the whole chapter follows is that any state holding a reference to another entity must be told when that entity goes away.
