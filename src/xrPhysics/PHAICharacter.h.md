# src/xrPhysics/PHAICharacter.h

> Declares the creature controller — a character that can be *asked to be
> somewhere* instead of being driven there.

**Needs** — [`PHAICharacter.cpp`](PHAICharacter.cpp.md) · [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md)
**Used by** — [`PHAICharacter.cpp`](PHAICharacter.cpp.md) · [`PHCharacter.cpp`](PHCharacter.cpp.md)
**Tier floor** — T2: a small specialization over the shared controller.

## Purpose

Declares the surface implemented in [`PHAICharacter.cpp`](PHAICharacter.cpp.md). Everything a
creature shares with the player is in [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md); what
is here is the small set of differences that follow from a creature being driven by a
pathfinder rather than by a hand on a keyboard.

## Exported units

- **`TryPosition`** — the defining one. Ask whether the creature can reach a given position
  by the next frame, and put it there if so. This is the AI's normal movement path; see the
  implementation for why it is not simply "set the position".
- **forced physics control** — a flag that makes the creature ignore the AI's requests and
  obey the solver alone; used while a creature is being thrown, is falling, or is otherwise
  not in charge of itself.
- **`Jump`** — unconditional, unlike the player's. A creature's brain has already decided it
  can jump.
- **`SetMaximumVelocity`** — plain assignment; the AI sets the speed it wants.
- **`InitContact`** — adds the creature-specific contact rules.
- **`ValidateWalkOn`** — delegates unchanged to the base; present as an override point only.
- **`Create`** — creates the body and clears the forced-control flag.
- **static damage suppression** — overridden to nothing: creatures take no collision damage
  from the world itself.

## Notes

A creature controller *has* a body in the solver and is stepped by it, but in the common case
its body is disabled and its position comes from the AI. The solver is the fallback for when
the AI's request cannot be honoured, not the primary driver. That inversion is the single
idea this class adds.
