# src/xrGame/ActorAnimation.h

> Names for the movement-state bit combinations the animation selector switches on.

**Needs** — [`actor_defs.h`](actor_defs.h.md) · [`ActorAnimation.cpp`](ActorAnimation.cpp.md)
**Used by** — [`ActorAnimation.cpp`](ActorAnimation.cpp.md)
**Tier floor** — T3: a naming convention over an existing bit set.

## Purpose

The animation selector in [`ActorAnimation.cpp`](ActorAnimation.cpp.md) is one large
decision over the actor's movement state, and the state is a bitset. This file names every
combination the selector actually distinguishes, so that the selector reads as a table of
named poses rather than as a wall of bitwise arithmetic.

It is a *documentation* file with no runtime content, and a rebuild will express the same
thing as enumerated cases or as a lookup table keyed by the bits. The value it carries is
the list itself: which combinations are distinguished at all.

## The distinguished combinations

Four movement directions — forward, backward, left strafe, right strafe — and the four
diagonal pairs of forward or backward with a strafe. Each of those eight exists in four
variants, giving thirty-two named states plus a plain jump:

- **walking** — the direction alone;
- **accelerated** — the direction with the acceleration bit, which is running;
- **crouched** — the direction with the crouch bit, which is creeping;
- **crouched and accelerated** — crouch-walking.

## Notes

**Two strafes at once are not a named state** and neither are forward-and-backward: the
input layer cancels opposing pairs before the selector ever sees them. A rebuild must keep
that cancellation upstream, or the selector's table acquires holes.

**Acceleration means "running" everywhere except while crouched**, where it means the
faster of the two crouch gaits. The same bit therefore selects a different *kind* of
faster, which is why the crouched variants need their own names rather than being composed
from the accelerated ones.
