# src/xrGame/SleepEffector.h

> Declares the sleep screen effect implemented in [`SleepEffector.cpp`](SleepEffector.cpp.md), and the authored record that describes one.

**Needs** — [`xrEngine/EffectorPP.h`](../xrEngine/EffectorPP.h.md) · [`xrEngine/Effector.h`](../xrEngine/Effector.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md)
**Used by** — [`ActorCameras.cpp`](ActorCameras.cpp.md) · [`SleepEffector.cpp`](SleepEffector.cpp.md)
**Tier floor** — T3: a declaration and one data record

## Purpose

Declares `CSleepEffectorPP` — the screen effect that covers sleep — and the small record a
caller fills to describe one. Substance is in [`SleepEffector.cpp`](SleepEffector.cpp.md).

```text
RECORD SleepEffectorDescription
  look         : ParameterSet    # the fully-asleep post-process look
  time         : real            # nominal lifetime
  time_attack  : real            # fade-in fraction
  time_release : real            # fade-out fraction
```

It also fixes the two **effector slot identifiers** this part of the game uses: one for
sleep and one for fatigue. Slot identifiers matter because the camera pipeline keeps at
most one effector per slot, so starting a second sleep effect replaces the first instead
of doubling the darkness. The two numbers are arbitrary but must not collide with any
other slot in the engine; they are part of a single flat numbering shared by every effect
in the game, which is the reason they are declared as fixed constants rather than
allocated.

Exported units:

- `CSleepEffectorPP` — the effect, with its four-phase state (beginning, faded in,
  sleeping, waking) exposed so that the actor's sleep logic can advance it.
- `Process` — the per-frame blend.
- `SSleepEffector` — the authored description above.
- The sleep and fatigue effector slot identifiers.
