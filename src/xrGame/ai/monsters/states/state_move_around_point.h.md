# src/xrGame/ai/monsters/states/state_move_around_point.h

> Declares a leaf state that would circle a point at a radius — dead twice over: nothing instantiates it, and it does not include its own implementation.

**Needs** — [`state.h`](../state.h.md) · [`state_data.h`](state_data.h.md) · [`state_move_to_point_inline.h`](state_move_to_point_inline.h.md)
**Used by** — [`state_move_around_point_inline.h`](state_move_around_point_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares an orbit-a-point leaf state holding a `StateMoveAroundPoint` record.

**This file is dead, in two independent ways, and a rebuilder should not carry it
forward.**

1. No creature's behaviour tree ever adds this state, under any identifier.
2. The declaration pulls in the *move-to-point* implementation file, not its own. Its
   sibling [`state_move_around_point_inline.h`](state_move_around_point_inline.h.md) is
   therefore included by nothing at all, and the state's methods have no definitions. It
   compiles only because nobody instantiates the template.

The behaviour the record describes — orbiting a position at a radius, used for a pack
circling prey — is not in the shipped game from this state. Where circling appears it comes
from elsewhere in the creature layer.

## Exported units

- **the orbit state** — declared entry, execution and completion test. None are defined.
