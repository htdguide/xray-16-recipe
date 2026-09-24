# src/xrPhysics/IClimableObject.h

> What a ladder must be able to answer about a character standing near it.

**Needs** — [`PHCharacter.h`](PHCharacter.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md)
**Used by** — [`ClimableObject.cpp`](../xrGame/ClimableObject.cpp.md) · [`ClimableObject.h`](../xrGame/ClimableObject.h.md) · [`ElevatorState.cpp`](ElevatorState.cpp.md) · [`ElevatorState.h`](ElevatorState.h.md) · [`IPHStaticGeomShell.h`](IPHStaticGeomShell.h.md) · [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md)
**Tier floor** — T3: pure geometric queries phrased as an interface.

## Purpose

A ladder is a game object, not a physics object, but the character controller's climbing
state machine ([`ElevatorState.cpp`](ElevatorState.cpp.md)) needs to reason about it every
step. Rather than teach physics what a ladder is, the ladder is asked a fixed set of
distance-and-direction questions. The abstraction is a **vertical strip in space** with an
axis, a side direction, a facing normal, and two endpoints.

Every query returns a *distance* and fills a *direction* — that pairing is the file's one
real decision. The state machine almost always wants both (how far, and which way to push),
and computing them separately would walk the ladder's geometry twice per step.

## `IClimableObject`

**Contract** — the ladder surface. An implementor provides:

- **Frame.** The climb axis (up along the ladder), the side direction (across its face) and
  the facing normal (out of its face), each as a direction plus the magnitude of the vector
  it came from.
- **Relation to a character.** Distance and direction to the ladder's axis; to its plane;
  to its lower endpoint; to its upper endpoint. Signed distances *along the axis* to the
  upper and lower endpoints, which is what tells the state machine when a climb has run out
  of ladder — these may be negative, meaning the character has passed the end.
- **Two predicates.** *Before the ladder* — the character is on the climbing side and
  within a caller-supplied tolerance of its face. *In touch* — the character is close enough
  to hold on at all.
- **Its surface material index**, so that while climbing, the character's footstep sounds
  and its last-touched material come from the ladder rather than from whatever is under it.
- **A downcast to the shell holder**, used only so that when the ladder object is destroyed
  the climbing character can drop its reference.

**Notes** — the interface carries no state and no notion of "attached": a character is never
*registered* with a ladder. It simply discovers the nearest one each step and keeps a
pointer until the distance exceeds a threshold. That is why the destroy path needs the
downcast — nothing else would tell the character its ladder is gone.
