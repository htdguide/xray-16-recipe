# src/xrGame/ActorCondition.h

> Declares the actor's condition model and the death sequence, both implemented in [`ActorCondition.cpp`](ActorCondition.cpp.md).

**Needs** — [`EntityCondition.h`](EntityCondition.h.md) · [`actor_defs.h`](actor_defs.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`ActorCondition_script.cpp`](ActorCondition_script.cpp.md) · [`ActorInput.cpp`](ActorInput.cpp.md) · [`Actor_Events.cpp`](Actor_Events.cpp.md) · [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`actor_mp_client.cpp`](actor_mp_client.cpp.md) · [`actor_script.cpp`](actor_script.cpp.md) · [`bloodsucker.cpp`](ai/monsters/bloodsucker/bloodsucker.cpp.md) · [`controller.cpp`](ai/monsters/controller/controller.cpp.md) · [`controller_psy_hit.cpp`](ai/monsters/controller/controller_psy_hit.cpp.md) · [`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md) · [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md) · _and 4 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CActorCondition`, the player-specific extension of the shared creature condition,
and `CActorDeathEffector`, the held-health death sequence. Substance is in
[`ActorCondition.cpp`](ActorCondition.cpp.md).

Exported units:

- `CActorCondition` — the condition. Its tuning fields are all read from one configuration
  section; its only public field is the walking-weight limit, which the movement code and
  the script layer both write.
- `LoadCondition` / `reinit` / `save` / `load` — the lifecycle: read tuning, reset to a new
  game's values, serialize into and out of a save.
- `UpdateCondition` / `UpdateBoosters` — the per-update integration.
- `ConditionJump` / `ConditionWalk` / `ConditionStand` — the three stamina costs of moving.
- `IsLimping` / `IsCantWalk` / `IsCantSprint` / `IsCantWalkWeight` — the four movement
  restrictions the movement code consults.
- `ApplyInfluence` / `ApplyBooster` — what a consumable does.
- The seventeen single-parameter boost entry points, one per boostable rate or immunity.
- `ChangeAlcohol` / `ChangeSatiety` / `GetSatiety` / `GetAlcohol` / `GetPsy` / `GetPower` —
  the scalar accessors the interface and the scripts read.
- `SetZoneDanger` / `GetZoneDanger` / `GetZoneMaxPower` — the anomaly-exposure readout.
- `PowerHit` / `ConditionHit` / `DisableSprint` / `PlayHitSound` / `HitSlowmo` — the damage
  policies.
- `WoundForEach` / `BoosterForEach` — script iteration over the live wound and boost sets,
  with an early-out when the callback returns true.
- `CActorDeathEffector` — the death sequence; holds the actor's health constant until its
  post-process effector finishes, then kills it.

## Notes

**The threshold latch set is declared here as a bitset and saved.** Nine bits, one per
tutorial announcement and one for the overload state. Saving them is what stops a tutorial
message from repeating after a reload, and it is why the bitset's *width* and bit order
are part of the save format.

**Three restriction latches are mutable and written from const queries.** A rebuild should
move the latch update into the per-frame update; see
[`ActorCondition.cpp`](ActorCondition.cpp.md).
