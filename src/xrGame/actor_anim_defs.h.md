# src/xrGame/actor_anim_defs.h

> The player character's animation vocabulary: which named motions exist, how they are grouped by posture and by weapon class, and where they come from in the model's animation bank.

**Needs** — [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`actor_defs.h`](actor_defs.h.md)
**Used by** — [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`ActorVehicle.cpp`](ActorVehicle.cpp.md)
**Tier floor** — T2: fixed-shape records of resolved motion handles; the resolution is a name lookup at load

## Purpose

The player's body is posed from a far larger motion set than any creature's, because it
must combine a *posture* (standing, crouching, climbing, sprinting), a *movement
direction*, and a *weapon-specific torso action* — and the torso half plays independently
of the legs so that the player can aim while walking. This file is the shape of that
combination: a nested set of records, each field one resolved motion handle, filled once
when the player's model is loaded by resolving names built from a base prefix.

Nothing here is behaviour. It is the declaration of *what animations a player model must
provide*, which makes it part of the frozen data contract: a model that lacks one of these
motions cannot be a player model.

## State

```text
RECORD DirectionalMotions          # one posture's four ground directions
  forward, backward, strafe_left, strafe_right : motion

RECORD WeaponTorsoMotions          # one weapon class's torso repertoire
  moving[idle, walk, run, sprint]  : motion    # torso carriage while the legs do that
  zoom, holster, draw, drop        : motion
  reload, reload_1, reload_2       : motion    # three variants: see notes
  attack, attack_zoom              : motion
  fire_idle, fire_end              : motion
  all_attack_0, all_attack_1, all_attack_2 : motion   # whole-body attacks, used only when standing still

RECORD PostureMotions              # one posture: standing, crouching, or climbing
  legs_idle, legs_turn             : motion
  jump_begin, jump_idle            : motion
  landing[2]                       : motion    # soft and hard landing
  death                            : motion
  walk, run                        : DirectionalMotions
  torso[13]                        : WeaponTorsoMotions   # indexed by weapon animation slot
  torso_idle, head_idle            : motion
  damage[12]                       : motion    # flinch reactions, indexed by damage direction/zone

RECORD SprintMotions               # sprinting has no torso half at all
  legs_forward, legs_strafe_left, legs_strafe_right           : motion
  legs_jump_forward, legs_jump_strafe_left, legs_jump_strafe_right : motion

RECORD ActorMotions
  dead_stop                        : motion
  normal, crouch, climb            : PostureMotions
  sprint                           : SprintMotions

RECORD VehicleAnimations           # the player's pose while driving
  idle_count                       : int          # 1..3 actually present
  idles[3]                         : motion
  steer_left, steer_right          : motion

RECORD ActorVehicleAnimations
  per_vehicle_type[2]              : VehicleAnimations
```

Invariants:

- The torso repertoire is indexed by **weapon animation slot**, of which there are
  thirteen. The count is frozen: the shipped player models supply exactly this many sets,
  and an item's configuration names its slot by index. It is not the number of inventory
  slots.
- The flinch reaction count is twelve, matching the damage-direction enumeration in
  [`actor_defs.h`](actor_defs.h.md); the two must agree or the arrays disagree with the
  index that addresses them.
- Sprinting has legs only. That is a design decision, not an omission: the player cannot
  aim or act while sprinting, so there is no torso half to blend.
- A vehicle collection declares how many idle variants were actually found, so a model may
  ship one, two or three and the selection picks among the present ones.

## `Create` (each record)

**Contract** — resolves every motion in the record against a loaded animated model, by
name, from one or two base prefixes supplied by the caller. Runs once when the player's
visual is bound. A missing motion yields an invalid handle rather than failing, so the
absence surfaces later as a pose that does not play.

**Invariants** — the naming convention is the contract with the art: names are composed by
concatenating the caller's prefix with a fixed suffix per field. The two-prefix forms
exist because a posture's legs and its torso are named from different roots. A rebuild may
choose any composition rule, but the *resulting names* must match the shipped models
exactly.

**Notes**

- The climbing posture is built by a separate entry point from the other two. Climbing
  reuses only part of the posture record — there is no crouching or weapon-class variation
  while on a ladder — so filling it through the general path would look for motions the
  models do not have.
- Three reload motions per weapon class allow a weapon to have distinct animations for a
  partial reload, an empty reload and a tactical reload; which one a weapon uses is
  decided by the weapon, not here.
- The whole-body attack variants exist because a melee swing that moves the legs cannot be
  expressed as a torso-only blend. They are selected only when the player is stationary,
  which is why the torso-only attack also exists.
