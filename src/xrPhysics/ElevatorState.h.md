# src/xrPhysics/ElevatorState.h

> Declares the per-character ladder-climbing state machine.

**Needs** — [`ElevatorState.cpp`](ElevatorState.cpp.md) · [`IElevatorState.h`](IElevatorState.h.md) · [`IClimableObject.h`](IClimableObject.h.md) · [`PHCharacter.h`](PHCharacter.h.md)
**Used by** — [`PHMovementControl.cpp`](../xrGame/PHMovementControl.cpp.md) · [`ElevatorState.cpp`](ElevatorState.cpp.md) · [`PHActorCharacter.cpp`](PHActorCharacter.cpp.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md) · [`PHSimpleCharacterInline.h`](PHSimpleCharacterInline.h.md)
**Tier floor** — T2: a state machine over geometric predicates.

## Purpose

Declares the surface implemented in [`ElevatorState.cpp`](ElevatorState.cpp.md). One of
these is embedded in every character controller; it is not separately created or destroyed.

## Exported units

- **the per-step tick** — evaluates the current state, may switch, and applies whatever
  force the current state owes.
- **`SetCharacter` / `SetElevator` / `Deactivate`** — binding and unbinding. Setting a
  ladder is a *proposal*: a nearer ladder wins, and a ladder beyond range is ignored.
- **state predicates** — active, near-down, near-either, climbing. The game layer branches
  animation and camera on these.
- **`GetControlDir`** — rewrites the character's desired movement direction into one the
  ladder permits, and reports whether the character may move at all. This is the
  desired-versus-permitted boundary for climbing.
- **`GetJumpDir`** — where a jump off the ladder goes.
- **`GetLeaderNormal`** — the ladder's facing normal, for orienting the character.
- **`ClimbDirection`** — signed: positive means the character is trying to go up.
- **`Depart`** — a deliberate let-go request from outside.
- **`UpdateMaterial`** — while climbing, the surface material is the ladder's, not the
  ground's.
- **`NetRelcase`** — drop the ladder reference when the ladder object is destroyed.

## Notes

The transition-inertia table declared here is a square matrix over the state enumeration,
holding a minimum distance and a minimum time for each transition. Its content, and the fact
that almost all of it is zero, are in the implementation.
