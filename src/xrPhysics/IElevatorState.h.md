# src/xrPhysics/IElevatorState.h

> The seven states a character can be in with respect to a ladder, published so
> the game layer can read them without seeing the state machine.

**Needs** — [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md)
**Used by** — [`ActorCameras.cpp`](../xrGame/ActorCameras.cpp.md) · [`ElevatorState.h`](ElevatorState.h.md)
**Tier floor** — T3: an enumeration and two entry points.

## Purpose

The climbing state machine lives in [`ElevatorState.cpp`](ElevatorState.cpp.md); the game
layer needs to know only *which* state a character is in, because animation selection,
camera behaviour and movement input all branch on it. This header is that published slice.

## `Estate`

**Contract** — the climbing state, in the order the machine uses it:

```text
ENUM Estate
  none           # a ladder is known but the character is unrelated to it
  near_up        # standing at the top of the ladder, able to step on
  near_down      # standing at the bottom, able to step on
  climbing_up    # on the ladder, moving up
  climbing_down  # on the ladder, moving down
  depart         # deliberately letting go
  no_ladder      # no ladder in range at all
```

`no_state` is the count, not a state, and is used to size the transition table.

**Notes** — the two `near_*` states exist because getting onto a ladder is the part that
goes wrong: a character who is merely close to a ladder must not have gravity switched off,
but must be ready to. Splitting "next to it" from "on it" is what lets the machine defer the
gravity change until the climb actually starts.

## `IElevatorState`

**Contract** — read the current state, and drop the ladder reference when the ladder object
is being destroyed. Nothing else is public: the machine is driven entirely from inside the
physics step.
