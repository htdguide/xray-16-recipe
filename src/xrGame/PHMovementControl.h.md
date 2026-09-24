# src/xrGame/PHMovementControl.h

> Declares the character-movement bridge, implemented in [`PHMovementControl.cpp`](PHMovementControl.cpp.md) and [`PHMovementDynamicActivate.cpp`](PHMovementDynamicActivate.cpp.md).

**Needs** — [`xrPhysics/PhysicsExternalCommon.h`](../xrPhysics/PhysicsExternalCommon.h.md) · [`xrPhysics/MovementBoxDynamicActivate.h`](../xrPhysics/MovementBoxDynamicActivate.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md)
**Used by** — [`ActivatingCharCollisionDelay.cpp`](ActivatingCharCollisionDelay.cpp.md) · [`Actor.h`](Actor.h.md) · [`ActorCameras.cpp`](ActorCameras.cpp.md) · [`ActorVehicle.cpp`](ActorVehicle.cpp.md) · [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`AmebaZone.cpp`](AmebaZone.cpp.md) · [`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md) · [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`GraviZone.cpp`](GraviZone.cpp.md) · [`HairsZone.cpp`](HairsZone.cpp.md) · [`NoGravityZone.cpp`](NoGravityZone.cpp.md) · [`PHMovementControl.cpp`](PHMovementControl.cpp.md) · [`PHMovementDynamicActivate.cpp`](PHMovementDynamicActivate.cpp.md) · [`PhysicsShellHolder.cpp`](PhysicsShellHolder.cpp.md) · _and 5 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CPHMovementControl`, the layer every walking creature moves through. Substance is in
[`PHMovementControl.cpp`](PHMovementControl.cpp.md); the careful posture switch is in
[`PHMovementDynamicActivate.cpp`](PHMovementDynamicActivate.cpp.md).

Three shape decisions are worth taking from the header before reading either.

**Two `Calculate` overloads, one body.** One takes an acceleration and a camera direction —
the player. One takes a path, a speed and a travel index — an AI. They are the two ways a
creature's mind can express movement, and everything below them is shared. A rebuild should
keep exactly these two, because a third would need its own answer to every question this file
resolves.

**Four collision boxes, not a capsule with a height.** Posture is a discrete choice among
authored boxes, with blending between them and a careful, failable switch. That is why
standing up under a ledge can fail rather than clip.

**Everything is mirrored.** Position, velocity, environment and the path state are all held
here as well as in the body, and the mirror is what path computations read. The body may be
stepping on another schedule.

Exported units — grouped:

- `CPHMovementControl` — the bridge. Also implements the physics layer's movement interface,
  so the solver can reach back into it without knowing about the game.
- `Calculate` (player form) / `Calculate` (path form) / `actor_calculate` — one frame of
  movement, from either kind of caller.
- `in_shedule_Update` — the scheduled poll; its only job is dropping a failed object capture.
- `PathNearestPoint` / `PathNearestPointFindUp` / `PathNearestPointFindDown` / `PathDIrLine` /
  `PathDIrPoint` / `CorrectPathDir` / `GetPathDir` / `SetPathDir` — path following: find where
  we are on the polyline, and which way that says to push. The up/down pair is the incremental
  search that makes a long path affordable.
- `MakeJumpPath` — replace the path mid-leap with a lateral correction toward the enemy: the
  auto-aim on a pouncing creature.
- `ActivateBox` / `ActivateBoxDynamic` / `InterpolateBox` / `SetBox` / `Box` / `Boxes` /
  `BoxID` — posture. The dynamic form can fail; the interpolating form is how crouching is
  animated.
- `Jump` (three forms) / `JumpV` / `JumpState` / `GetJumpParam` / `GetJumpMinVelParam` /
  `JumpMinVelTime` / `SetJumpUpVelocity` / `JumpType` — trajectory solving, plus the
  three-way classification (rises only, apex at the target, rises and falls) the animation
  layer needs.
- `Environment` / `OldEnvironment` / `SetEnvironment` / `GroundNormal` / `EEnvironment` —
  on ground, at a wall, or in the air, kept for this frame and the last so a landing is a
  detectable transition.
- `gcontact_Was` / `gcontact_Power` / `gcontact_HealthLost` / `GetContactSpeed` /
  `SetCrashSpeeds` / `BlockDamageSet` / `CollisionDamageInfo` / `ContactBone` — the fall and
  collision damage model: a linear ramp between two configured speeds, suppressible for a
  number of physics steps after a discontinuous move.
- `ApplyImpulse` / `ApplyHit` / `AddControlVel` / `vExternalImpulse` / `bExernalImpulse` —
  pushes from outside, and how a hit interrupts movement.
- `PHCaptureObject` (two forms) / `PHReleaseObject` / `PHCapture` /
  `PHCaptureGetNearestElemPos` / `PHCaptureGetNearestElemTransform` — grabbing and carrying a
  physical object, with the grab point chosen by proximity.
- `TryPosition` / `SetPosition` / `GetPosition` / `GetCharacterPosition` /
  `InterpolatePosition` / `GetDesiredPos` / `GetDeathPosition` / `VirtualMoveTo` /
  `b_exect_position` — placement, and the probe that answers "where would walking there put
  me" without moving anything.
- `SetVelocity` / `GetVelocity` / `SetCharacterVelocity` / `GetCharacterVelocity` /
  `GetSmoothedVelocity` / `GetVelocityMagnitude` / `GetVelocityActual` /
  `GetXZVelocityActual` / `GetActVelProj` / `GetActVelInGoingDir` / `GetXZActVelInGoingDir` /
  `SetVelocityLimit` / `VelocityLimit` — velocity in every projection the animation and
  gameplay layers ask for. The horizontal-only and along-heading forms exist because the
  locomotion blend is driven by forward speed on the ground plane, not by total speed.
- `CreateCharacter` / `DestroyCharacter` / `AllocateCharacterObject` / `DeleteCharacterObject`
  / `CharacterExist` / `IsCharacterEnabled` / `EnableCharacter` / `DisableCharacter` /
  `Freeze` / `UnFreeze` / `CharacterType` — the body's lifecycle. Allocation picks the kind;
  creation builds it. A character may exist with no body.
- `SetMass` / `GetMass` / `SetMaterial` / `SetApplyGravity` / `SetAirControlParam` /
  `SetFrictionFactor` / `GetFrictionFactor` / `MulFrictionFactor` / `FootRadius` /
  `CollisionEnable` — the body's physical parameters.
- `SetRestrictionType` / `SetActorRestrictorRadius` / `SetActorMovable` / `UpdateObjectBox` —
  navigation restriction, including telling another character how much room this one needs
  from a particular angle.
- `SetForcedPhysicsControl` / `ForcedPhysicsControl` / `PhysicsOnlyMode` /
  `SetNonInteractive` — modes in which the body, or something else entirely, owns the
  placement.
- `TraceBorder` / `BorderTraceCallback` / `isOutBorder` / `setOutBorder` /
  `in_dead_area_count` — the signed count of injurious volumes entered, maintained by ray
  tracing each frame's movement and testing which way each crossed surface faced.
- `update_last_material` / `injurious_material_idx` / `SetPLastMaterialIDX` — what is
  underfoot.
- `SetOjectContactCallback` / `SetFootCallBack` / `ObjectContactCallback` — contact
  notification, with the feet kept separate from the body so footstep sounds can be chosen by
  the material under the foot.
- `GetSyncItem` — the body as a network-synchronizable item.
- `ElevatorState` — the state machine for riding a ladder or a lift.
- `NetRelcase` — drop a reference to an object the engine is destroying, in particular a
  captured one.

## Notes

`CalcMaximumVelocity` is declared twice, with different signatures, and both bodies are empty.
Neither is called.

The three friction constants at the top of the implementation — ground, air and wall — are the
remains of a friction model that now lives entirely in the body. The default box half-extents
(0.35 by 0.8 by 0.35) are a fallback for a creature whose configuration declares no boxes.

`path_few_point` is declared as ten and never used.
