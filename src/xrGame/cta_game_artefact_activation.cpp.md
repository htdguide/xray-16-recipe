# src/xrGame/cta_game_artefact_activation.cpp

> The activation sequence for a capture-the-artefact objective: the same staged timeline as a normal artefact, with the visual effects and the self-destruction removed.

**Needs** — [`cta_game_artefact_activation.h`](cta_game_artefact_activation.h.md) · [`artefact_activation.h`](artefact_activation.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`Level.h`](Level.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a timed state advance driven by the frame delta

## Purpose

A normal artefact, when activated, runs a staged timeline — charge, flash, spawn an anomaly,
be consumed — with a particle and sound effect per stage. The capture-the-artefact objective
must run *something* so that activation takes time and can be interrupted, but it must not
be consumed (the artefact has to reappear at its base) and it must not be dressed up with
effects the mode has no art for.

This subclass therefore keeps the timeline and strips two things: the per-stage effect
change, and the destruction at the end. Every other step delegates upward.

## State

`Stateless.` The stage index, the time in the current stage and the per-stage durations all
live in the base activation record.

## `UpdateActivation`

**Contract** — advances the activation timeline by one frame. Does nothing when no
activation is running. Must not be called while the physics world is stepping, because the
stage change it can trigger spawns an entity. Advances one stage at a time; past the last
stage it returns to the idle stage rather than destroying the artefact. Spawns the anomaly
only on the authoritative side, and only on entry to the spawn stage.

```text
FUNCTION update_activation()
  IF not in progress: RETURN
  state_time = state_time + frame_delta
  IF state_time >= duration_of(current_stage)
    current_stage = next stage
    IF current_stage is past the last
      current_stage = idle               # see Notes
    state_time = 0
    change_effects()                     # overridden to nothing
    IF current_stage is the spawn stage AND authoritative
      spawn_anomaly()
  update_effects()
```

**Notes** — the difference from the base sequence is the wrap. The base class, on running
past the last stage, stops the artefact's physics participation and destroys the object;
here both of those are removed and the stage index simply returns to idle. That is what
makes the objective artefact survivable: it has been "used", the anomaly (if any) has been
spawned, and the artefact itself is still there to be teleported home by
[`cta_game_artefact.cpp`](cta_game_artefact.cpp.md).

The anomaly spawn is gated on the authoritative side because it creates an entity, and
entities are created in exactly one place; every client receives it as a spawn message.

The assertion that the physics world is not currently stepping is the load-bearing form of a
real constraint: this update can create and destroy entities, and doing so from inside a
physics step would mutate the body set the solver is iterating. A rebuild needs the
constraint even if it expresses it differently.

## `ChangeEffects`

**Contract** — deliberately empty. The base class would switch the particle and sound effect
to the one authored for the new stage; this mode has none, so the stage change is silent.

## `Load` / `Start` / `Stop` / `UpdateEffects` / `SpawnAnomaly` / `PhDataUpdate`

**Contract** — pure delegation to the base activation, each one. They exist only so that the
class declares the whole surface as overridable; a rebuild can omit every one of them.
