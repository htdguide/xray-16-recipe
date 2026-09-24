# src/xrGame/Actor.h

> Declares the player's entity and the full set of behaviours mixed into it.

**Needs** — [`Actor_Flags.h`](Actor_Flags.h.md) · [`actor_defs.h`](actor_defs.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`PhraseDialogManager.h`](PhraseDialogManager.h.md) · [`step_manager.h`](step_manager.h.md) · [`fire_disp_controller.h`](fire_disp_controller.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`game_news.h`](game_news.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`ActorBackpack.cpp`](ActorBackpack.cpp.md) · [`ActorCameras.cpp`](ActorCameras.cpp.md) · [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`ActorEffector.cpp`](ActorEffector.cpp.md) · [`ActorHelmet.cpp`](ActorHelmet.cpp.md) · [`ActorInput.cpp`](ActorInput.cpp.md) · [`ActorMountedWeapon.cpp`](ActorMountedWeapon.cpp.md) · [`ActorVehicle.cpp`](ActorVehicle.cpp.md) · [`Actor_Events.cpp`](Actor_Events.cpp.md) · [`Actor_Feel.cpp`](Actor_Feel.cpp.md) · [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`Actor_Network.cpp`](Actor_Network.cpp.md) · _and 140 more_
**Tier floor** — T2: a declaration.

## Purpose

Declares the surface implemented across [`Actor.cpp`](Actor.cpp.md) and the twelve
`Actor*` files beside it. Its own load-bearing content is the **mix-in list**, which is the
map a reader needs before any of those files makes sense. The actor is, in one object:

- a **living entity** — the spine described in the [chapter opener](README.md): game object
  → collidable/renderable object → entity → living entity;
- an **input receiver** — it sits on the input stack and reads actions, never keys;
- a **touch sense** and a **sound sense** — two of the three feel interfaces;
- an **inventory owner** — slots, belt, rucksack, trade, character info;
- a **dialogue manager** — it can hold a conversation and receives phrase graphs;
- a **step manager** — footstep sounds and their material-dependent selection.

Everything else it holds is composed rather than inherited: a camera set, an effector
manager, a condition object, a perception (memory) manager, a location manager, a physics
support object, a statistics manager, and the registry wrappers for encyclopedia and news.

## Exported units

- `CActor` — the class itself. Its methods are contracted in
  [`Actor.cpp`](Actor.cpp.md) and the sibling `Actor*` twins:
  animation in [`ActorAnimation.cpp`](ActorAnimation.cpp.md), cameras in
  [`ActorCameras.cpp`](ActorCameras.cpp.md), input in [`ActorInput.cpp`](ActorInput.cpp.md),
  movement in [`Actor_Movement.cpp`](Actor_Movement.cpp.md), networking in
  [`Actor_Network.cpp`](Actor_Network.cpp.md), events in
  [`Actor_Events.cpp`](Actor_Events.cpp.md), senses in
  [`Actor_Feel.cpp`](Actor_Feel.cpp.md), weapon interaction in
  [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md), the vehicle seat in
  [`ActorVehicle.cpp`](ActorVehicle.cpp.md) and the mounted weapon in
  [`ActorMountedWeapon.cpp`](ActorMountedWeapon.cpp.md).
- `isActorAccelerated(movement_state, aiming)` — the one predicate that decides whether a
  given movement state counts as "moving fast", used by the animation selector, the
  collision-box selector and the motion icon alike. Implemented in
  [`Actor_Movement.cpp`](Actor_Movement.cpp.md).
- `Actor()` and the actor global — the single live actor, reachable from anywhere.
- `g_start_position`, `g_start_game_vertex_id` — where a new game places the actor, set by
  the level start path.
- `s_fFallTime` — the authored duration of a fall's landing recovery.

## Notes

**Casting is how the rest of the game asks "what is this object".** The actor answers a
family of cast queries — *are you an actor*, *an inventory owner*, *an attachment owner*,
*an input receiver*, *a game object* — each returning itself. A rebuild should read these
as an explicit capability query on the object interface, not as a language feature: the
list of questions *is* the set of roles the engine knows about, and it appears again on
every other entity in the chapter.

**Two update rates are declared here and the choice is per-object.** The actor implements
both the unconditional per-frame update and the budgeted scheduled update, and declares
that it never registers with the scheduler's degrade path — see
[`xrSheduler.cpp`](../xrEngine/xrSheduler.cpp.md) for what that opts out of.
