# src/xrGame/CharacterPhysicsSupport.h

> Declares the alive-capsule / dead-ragdoll boundary implemented in [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md).

**Needs** — [`PHSkeleton.h`](PHSkeleton.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`PHSoundPlayer.h`](PHSoundPlayer.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`character_hit_animations.h`](character_hit_animations.h.md) · [`death_anims.h`](death_anims.h.md) · [`character_shell_control.h`](character_shell_control.h.md) · [`animation_utils.h`](animation_utils.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — [`ActivatingCharCollisionDelay.cpp`](ActivatingCharCollisionDelay.cpp.md) · [`Actor.cpp`](Actor.cpp.md) · [`ActorCameras.cpp`](ActorCameras.cpp.md) · [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`ActorInput.cpp`](ActorInput.cpp.md) · [`ActorMountedWeapon.cpp`](ActorMountedWeapon.cpp.md) · [`ActorVehicle.cpp`](ActorVehicle.cpp.md) · [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`AmebaZone.cpp`](AmebaZone.cpp.md) · [`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md) · [`Car.cpp`](Car.cpp.md) · [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · _and 22 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares the physical half of a living creature: which of the two representations exists,
the eight subsystems that hang off it, and the `in_*` family that the owning entity's
lifecycle calls through. Substance in
[`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md).

The naming convention is worth stating once: every method prefixed `in_` is a hook the
*entity* calls from the corresponding point in its own lifecycle —
`in_NetSpawn`, `in_UpdateCL`, `in_shedule_Update`, `in_Hit`, `in_Die`, `in_NetDestroy`,
`in_NetSave`, `in_NetRelcase`, `in_ChangeVisual`, `in_Load`, `in_Init`. Reading those in
order gives the whole lifecycle.

## The two enumerations

```text
EType   : actor | stalker | biting
          # decides the character shape, the movement restriction class,
          # whether the player can push this creature, whether the weapon
          # graft and hit animations apply, and whether the skeleton is
          # re-rooted at the pelvis (everything but "biting" is).

EState  : alive | dead | removed
          # alive = a character capsule exists; dead = a ragdoll exists;
          # removed = neither, the corpse is being taken out of the world.
```

## The five flags

```text
death_anim_on           # the corpse idle cycle has been started
skeleton_in_shell       # which pointer owns the built shell
specific_bonce_demager  # this creature has its own collision-damage factor
block_hit               # refuse hits for 2 seconds after a free-ragdoll death
use_hit_anims           # play a flinch animation when struck
```

## Exported units

**Lifecycle hooks** — `in_Load`, `in_Init`, `in_NetSpawn`, `in_UpdateCL`,
`in_shedule_Update`, `in_Hit`, `in_Die`, `in_ChangeVisual`, `in_NetSave`, `in_NetDestroy`,
`in_NetRelcase`.

**Representation**

- `movement` — the character controller, while alive.
- `CreateCharacter`, `CreateCharacterSafe` — build the capsule, the safe form
  collision-correcting first.
- `SpawnInitPhysics`, `SpawnCharacterCreate` — the fork at spawn between capsule and
  ragdoll.
- `CreateSkeleton`, `CreateShell`, `ActivateShell`, `EndActivateFreeShell`, `KillHit` —
  private: the death transition, in pieces.
- `FlyTo` — private: step the solver by hand to drive a fresh ragdoll back into place.
- `CollisionCorrectObjPos` — place something without it intersecting the world.
- `set_movement_position`, `ForceTransform` — teleport a living creature.
- `SetRemoved`, `IsRemoved`, `CanRemoveObject` — taking a corpse out of the world; the
  player never leaves it.

**Animation coupling**

- `on_create_anim_mov_ctrl`, `on_destroy_anim_mov_ctrl` — put the character controller
  into and out of non-interactive mode while an animation moves the creature.
- `run_interactive`, `update_interactive_anims`, `is_interactive_motion` — animations that
  want collision callbacks, and the death animation driving a ragdoll.
- `create_animation_collision`, `update_animation_collision`, `destroy_animation_collision`,
  `animation_collision` — a temporary shell giving an animated creature collision, with a
  three-second self-destruct that is pushed forward on each request.
- `UpdateDeathAnims` — private: start the corpse idle cycle once nothing drives the
  ragdoll.
- `DeathAnimCallback` — private, static: the death-animation completion hook.
- `set_use_hit_anims` — enable or suppress flinch animations.
- `can_drop_active_weapon` — true only once the corpse has gone limp.

**The weapon graft**

- `AddActiveWeaponCollision`, `RemoveActiveWeaponCollision` — private: merge a held
  weapon's collision into the corpse's hand bone and separate it back out with the right
  velocity.
- `has_shell_collision_place`, `on_child_shell_activate` — the notification by which a
  weapon becoming a body of its own triggers the separation.
- `bone_chain_disable`, `bone_fix_clear` — private: pin the animated bones that now carry
  physical geometry.

**Other**

- `ph_sound_player` — collision and footstep sound.
- `ik_controller`, `CreateIKController`, `DestroyIKController` — foot placement, alive
  only.
- `get_collision_hit_callback`, `set_collision_hit_callback` — who is told about damage
  from physical collisions.
- `PHGetLinearVell` — velocity, from whichever representation is live.
- `PHGetSyncItemsNumber`, `PHGetSyncItem` — one synchronized body while alive, the whole
  per-bone shell state once dead.
- `IsSpecificDamager`, `BonceDamageFactor` — this creature's collision-damage scaling.
- `DoCharacterShellCollide` — private: a stalker who died wounded does not collide with
  the living.
