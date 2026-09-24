# src/xrPhysics/PHSimpleCharacter.h

> Declares the concrete character controller every creature and the player share: a four-shape capsule model, a large set of one-bit contact conclusions, and the collision-damage record.

**Needs** — [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · [`PHSimpleCharacterInline.h`](PHSimpleCharacterInline.h.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`ElevatorState.h`](ElevatorState.h.md) · [`IColisiondamageInfo.h`](IColisiondamageInfo.h.md) · [`Physics.h`](Physics.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`PHAICharacter.cpp`](PHAICharacter.cpp.md) · [`PHAICharacter.h`](PHAICharacter.h.md) · [`PHActorCharacter.cpp`](PHActorCharacter.cpp.md) · [`PHActorCharacter.h`](PHActorCharacter.h.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · [`PHSimpleCharacterInline.h`](PHSimpleCharacterInline.h.md)
**Tier floor** — T1: it owns solver shapes and bodies directly and queries the collision database per step.

## Purpose

Declares the surface implemented in [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) and
[`PHSimpleCharacterInline.h`](PHSimpleCharacterInline.h.md). The abstract contract it satisfies is in
[`PHCharacter.h`](PHCharacter.h.md); this is the only implementation, specialized further by the
player and AI variants only in collision filtering and restriction policy.

Two things in this declaration are decisions rather than plumbing and deserve to be read here: the
*shape model* — a character is four shapes, not one capsule — and the *state vocabulary*, a
deliberately large set of one-bit conclusions that the controller draws from contacts each step and
then reasons over. A rebuild that reduces either loses behaviour.

## the shape model

```text
RECORD CharacterShapes
  wheel  : sphere at the character's feet, radius r       # the ground-riding shape
  shell  : cylinder, radius r/1.2, spanning the torso     # the body
  hat    : sphere at head height, radius r/1.2            # the head
  cap    : sphere at 2.5r high, radius 2r, STATIC-BLIND   # the path probe, collides only
                                                          # with dynamic objects
```

**Invariants** — the wheel is the full radius; the shell and hat are narrower by a fixed factor of
1.2. That inset is what makes step climbing work: the widest part of the character is at the feet,
so a shin-high obstacle meets the sphere — which rolls over it — rather than the cylinder, which
would jam. The cylinder is lowered by an amount derived from the same factor so that it meets the
sphere tangentially rather than leaving a ledge.

The cap is not part of the character's body at all. It is a probe: it is told not to collide with
the static world, and its only contact callback sets one flag. See
[`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md).

## the state vocabulary

```text
RECORD ControllerState
  # --- what happened this step, filled by the contact callback ---
  is_contact, any_contacts   : bool     # touched anything at all
  side_contact               : bool     # touched something at torso height, or the probe fired
  valide_ground_contact      : bool     # a contact was chosen as "the ground"
  valide_wall_contact        : bool     # a contact was chosen as "the wall"
  ground_contact_normal, _position : vector
  wall_contact_normal,   _position : vector
  contact_count              : int
  on_object                  : bool     # standing on a dynamic body, not the level
  friction_factor            : real     # the best friction any contact offered

  # --- the previous step's answers, for edge detection ---
  was_contact, was_control, was_side_contact, was_on_object : bool

  # --- the conclusions the controller reasons over ---
  is_control    : bool    # the game is asking this character to move
  meet, depart  : bool    # the rising and falling edges of contact
  meet_control  : bool    # a one-shot latch: control was regained (drives footstep sounds)
  stop_control, depart_control : bool
  on_ground     : bool
  lose_ground   : bool    # no good ground: animation should play a falling state
  lose_control  : bool    # BALLISTIC: the intent is ignored, the body just flies
  jump, jumping : bool
  clamb_jump    : bool    # climbing a step or ledge
  death_pos     : bool    # the body is being pushed out of geometry; hold a safe position

  # --- externally applied motion ---
  external_impulse   : bool
  ext_impuls_stop_step : int (64-bit)
  ext_imulse         : vector
  jump_accel         : vector
  jump_depart_position, depart_position, clamb_depart_position, death_position : vector

  # --- parameters ---
  radius, cyl_hight, mass, max_velocity, jump_up_velocity : real
  air_control_factor, collision_damage_factor             : real
  acceleration, cam_dir  : vector
  non_interactive        : bool
```

**Invariants** — the three that matter:

- **`lose_control` is the controller's master switch.** While it is set the character is a
  projectile: its intent is scaled away, its speed limit changes, and its animation goes to a
  falling state. Everything the controller does is conditioned on it.
- **Every per-step conclusion is cleared at the start of the post-solve pass and refilled by the
  contact callback during the next collision phase.** A conclusion read outside that window is stale
  by one step.
- **`was_*` is written from `is_*` before the clear**, which is the only way edges are detected.

## the collision-damage record

```text
RECORD CollisionDamageInfo
  damage_contact   : the contact that did it
  contact_velocity : real     # an EFFECTIVE velocity, not a real one; read-once (see the notes)
  hit_callback     : the hitting object's own damage hook
  obj_id           : int      # the object that hit, or none for the level
  dmc_signum       : real     # ±1: which way the contact normal points relative to this character
  dmc_type         : one of {static, object}
  hit_type         : the damage category
  is_initiated     : bool
```

**Invariants** — only the **worst** contact of a step is kept: each candidate replaces the stored one
only if its effective velocity is larger. Reading the velocity **clears it** — it is a one-shot
handoff to the game layer, and reading it twice in a frame gets zero the second time.

## Exported units

**Lifecycle** — `Create` (from a box of sizes), `SetBox` (resize in place), `Destroy`, `Enable`,
`Disable`, `EnableObject`, `IsEnabled`, `SetNonInteractive`.

**Step participation** — `PhTune`, `PhDataUpdate`, `InitContact`, `Collide`, `get_spatial_params`,
`dSpace`, `dSpacedGeom`, `Freeze`, `UnFreeze`, `step`, `collision_enable`, `collision_disable`.

**Intent** — `SetAcceleration`, `GetAcceleration`, `ControlAccel`, `SetCamDir`, `CamDir`,
`SetMaximumVelocity`, `GetMaximumVelocity`, `SetJupmUpVelocity`, `JumpState`,
`SetAirControlFactor`, `AddControlVel`.

**Environment** — `CheckInvironment`, `GroundNormal`, `ContactWas`, `TouchRestrictor`,
`UpdateRestrictionType`, `FrictionFactor`, `ElevatorState`, `SetElevator`,
`ValidateWalkOn`, `ValidateWalkOnMesh`, `ValidateWalkOnObject`.

**Placement** — `SetPosition`, `GetPosition`, `GetBodyPosition`, `BodyPosition`,
`GetPreviousPosition`, `IPosition`, `DeathPosition`, `FootRadius`, `get_Box`.

**Motion** — `GetVelocity`, `SetVelocity`, `GetSmothedVelocity`, `ApplyImpulse`, `ApplyForce` in
three forms, `SetMas`, `Mass`.

**Contacts and callbacks** — `SetObjectContactCallback`, `SetObjectContactCallbackData`,
`SetWheelContactCallback`, `SetStaticContactCallBack`, `ObjectContactCallBack`,
`SwitchOFFInitContact`, `SwitchInInitContact`, `SetMaterial`, `update_last_material`,
`SetPhysicsRefObject`, `NetRelcase`.

**Damage** — `CollisionDamageInfo` in both forms, and the record's own surface: `ContactVelocity`,
`HitDir`, `HitPos`, `HitType`, `SetHitType`, `HitCallback`, `DamageInitiator`, `DamageInitiatorID`,
`ContactBone`, `SetInitiated`, `IsInitiated`, `GetAndResetInitiated`, `Reinit`,
`SetCollisionDamageFactor`.

**Network** — `get_State`, `set_State`.

## Notes

`ignore_material` — a material flagged as an *actor obstacle* is skipped by the material-picking ray
that decides what the character is standing on. Those are invisible blockers placed by level authors
to keep creatures out of places; they must stop movement but must not become the surface the
footstep sound is chosen from.

`def_spring_rate` (a half) and `def_dumping_rate` (roughly twenty) are the softness the character
imposes on every contact it takes part in. They are tuning constants with no derivation; see
[`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) for what they buy.
