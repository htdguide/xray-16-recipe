# src/xrGame/PhysicsShellHolder.h

> Declares the physics-owning game object base and the adapter interface it presents to the dynamics module, implemented in [`PhysicsShellHolder.cpp`](PhysicsShellHolder.cpp.md).

**Needs** — [`GameObject.h`](GameObject.h.md) · [`ParticlesPlayer.h`](ParticlesPlayer.h.md) · [`xrEngine/IObjectPhysicsCollision.h`](../xrEngine/IObjectPhysicsCollision.h.md) · [`xrPhysics/IPhysicsShellHolder.h`](../xrPhysics/IPhysicsShellHolder.h.md)
**Used by** — [`AmebaZone.cpp`](AmebaZone.cpp.md) · [`Artefact.cpp`](Artefact.cpp.md) · [`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md) · [`BlackGraviArtifact.h`](BlackGraviArtifact.h.md) · [`BreakableObject.cpp`](BreakableObject.cpp.md) · [`BreakableObject.h`](BreakableObject.h.md) · [`CarWeapon.cpp`](CarWeapon.cpp.md) · [`ClimableObject.cpp`](ClimableObject.cpp.md) · [`ClimableObject.h`](ClimableObject.h.md) · [`Entity.cpp`](Entity.cpp.md) · [`Entity.h`](Entity.h.md) · [`ExplosiveRocket.cpp`](ExplosiveRocket.cpp.md) · [`GraviZone.cpp`](GraviZone.cpp.md) · [`HairsZone.cpp`](HairsZone.cpp.md) · _and 34 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `CPhysicsShellHolder` — a game object that also plays particles, also answers the
engine's object-collision interface, and also answers the physics module's shell-holder
interface. Substance is in [`PhysicsShellHolder.cpp`](PhysicsShellHolder.cpp.md).

The declaration's own contribution is the *default answers*: a long list of "which
optional capability do you have" queries that the base answers with nothing, so that a
caller holding any shell holder can ask every question without knowing what it holds.
Subclasses override the ones they can answer. In a rebuild these are capability lookups on
a component set, not virtual methods.

Exported units:

- `CPhysicsShellHolder` — the base class; owns the shell and the collision sound player.
- Shell acquisition and teardown: `create_physic_shell`, `activate_physic_shell`,
  `setup_physic_shell`, `deactivate_physics_shell`, `correct_spawn_pos`.
- Lifecycle: `net_Spawn`, `net_Destroy`, `save`, `load`, `UpdateCL`, `OnChangeVisual`,
  `init`.
- Scheduler control: `SheduleRegister`, `SheduleUnregister`, `IsSheduled`,
  `register_schedule`.
- Body queries and commands: `PPhysicsShell`, `physics_shell` (const and mutable),
  `physics_character`, `physics_collision`, `PHGetLinearVell`, `PHSetLinearVell`,
  `PHSetMaterial` (by name and by index), `GetMass`, `PHFreeze`, `PHUnFreeze`,
  `EffectiveGravity`, `PHGetSyncItemsNumber`, `PHGetSyncItem`.
- Pose serialization: `PHSaveState`, `PHLoadState`.
- Damage: `Hit`, `PHHit`.
- Capability queries defaulting to nothing: `ph_destroyable`,
  `PHCollisionDamageReceiver`, `PHSkeleton`, `cast_IDamageSource`,
  `character_physics_support`, `character_ik_controller`, `get_collision_hit_callback`,
  `set_collision_hit_callback`, `enable_notificate`, `ActivationSpeedOverriden`.
- Capability queries answering itself: `PhysicsShellHolder`, `cast_physics_shell_holder`,
  `cast_particles_player`, `ph_sound_player`.
- Parent/child shell coordination: `has_shell_collision_place`, `on_child_shell_activate`,
  `on_physics_disable`.
- The physics-module adapter surface, private to the interface it implements:
  `ObjectXFORM`, `ObjectPosition`, `ObjectName`, `ObjectNameVisual`, `ObjectNameSect`,
  `ObjectGetDestroy`, `ObjectGetCollisionHitCallback`, `ObjectID`, `IObject`,
  `ObjectCollisionModel`, `ObjectKinematics`, `ObjectCastIDamageSource`,
  `ObjectProcessingActivate`, `ObjectProcessingDeactivate`, `ObjectSpatialMove`,
  `ObjectPPhysicsShell`, `has_parent_object`, `PHCapture`, `IsInventoryItem`, `IsActor`,
  `IsStalker`, `IsCollideWithBullets`, `IsCollideWithActorCamera`, `HideAllWeapons`,
  `MovementCollisionEnable`, `ObjectPhSoundPlayer`, `ObjectPhCollisionDamageReceiver`,
  `BonceDamagerCallback`.
- `dump` — debug-only textual rendering of the object.
