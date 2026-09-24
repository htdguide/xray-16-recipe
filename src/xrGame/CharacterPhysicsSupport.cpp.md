# src/xrGame/CharacterPhysicsSupport.cpp

> The transition from a walking character to a falling body: while alive the creature is an upright capsule steered by its movement controller, and on death it becomes a ragdoll built from its own skeleton — with an optional death animation driving the joints on the way down.

**Needs** — [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`PHSoundPlayer.h`](PHSoundPlayer.h.md) · [`character_hit_animations.h`](character_hit_animations.h.md) · [`death_anims.h`](death_anims.h.md) · [`character_shell_control.h`](character_shell_control.h.md) · [`IKLimbsController.h`](IKLimbsController.h.md) · [`imotion_position.h`](imotion_position.h.md) · [`imotion_velocity.h`](imotion_velocity.h.md) · [`interactive_animation.h`](interactive_animation.h.md) · [`animation_movement_controller.h`](animation_movement_controller.h.md) · [`ActivatingCharCollisionDelay.h`](ActivatingCharCollisionDelay.h.md) · [`Actor.h`](Actor.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`Inventory.h`](Inventory.h.md) · [`Hit.h`](Hit.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md); callers name that, not this file.
**Tier floor** — T1: it drives the rigid-body library directly, including stepping the world by hand

## Purpose

Every living creature has two mutually exclusive physical representations, and this file
owns the boundary between them.

**Alive**: a *character controller* — an upright capsule that is pushed around by a
movement controller, does not tip over, climbs steps and is what collision against the
world actually uses. The skeleton is purely animated; it has no physical existence.

**Dead**: a *shell* — one rigid body per skeleton element, jointed together, simulated by
the rigid-body library. The animation stops driving the bones and the bones start driving
the animation.

The transition is the hard part, and it is hard for a reason a rebuild will hit too: the
body must appear exactly where the character was, at the velocity the character had, with
the pose the animation had reached, without being born intersecting the world. Four
separate mechanisms exist for that, and together they are most of the file:

1. **collision-corrected placement** — before a body or a capsule is created, an activation
   shape is expanded at the intended position and pushed out of whatever it overlaps, and
   the entity is moved to the result;
2. **fly-back** — having been pushed out, the ragdoll is then *driven* back toward where the
   character actually was, by stepping the physics world by hand ten times with gravity
   disabled. This is the only place in the game layer that steps the solver itself;
3. **death animations** — instead of going limp, the ragdoll may be driven by a chosen
   animation for the first moments, so a shot creature staggers rather than collapsing;
4. **the weapon graft** — a stalker's held weapon is merged *into* the corpse's own shell
   as extra collision geometry on the hand bone, so the corpse does not fall through its
   own rifle, and is separated back out if the weapon is ever activated on its own.

## State

```text
ENUM CharacterType   = actor | stalker | biting     # biting = a non-humanoid mutant
ENUM CharacterState  = alive | dead | removed

RECORD CharacterPhysicsSupport
  type, state
  movement_control     : the upright capsule and its steering
  physics_skeleton     : a prebuilt but inactive shell, held ready
  physics_shell        : the active ragdoll (borrowed reference into the entity)
  sound_player         : collision and footstep sounds
  ik_controller        : optional foot placement, while alive
  hit_animations       : the flinch-on-being-hit animation set
  death_anims          : the table mapping a killing blow to a death animation
  shell_control        : friction, joint resistance and the fatal impulse
  interactive_motion   : the death animation currently driving the ragdoll
  interactive_animation: an animation whose collision the world must respect
  animated_collision   : a temporary shell so animated bones collide
  weapon_geoms         : the grafted weapon collision, and the bone it hangs on
  active_item_obj      : whose weapon that was
  saved_hit            : the last hit received, and how long it stays valid
  flags                : death_anim_on, skeleton_in_shell, specific_bounce_damager,
                         block_hit, use_hit_anims
```

Invariants:

- **at most one representation exists at a time.** A live creature has a character and no
  shell; a dead one has a shell and no character. Both being absent is legal (the removed
  state); both present is not.
- the `skeleton_in_shell` flag records which pointer owns the built shell — the held-ready
  skeleton or the active shell — because the two swap by pointer transfer and only one
  must be deleted.
- a ragdoll is built with its *pelvis* as the skeleton root, not the animation root, for
  every creature except the non-humanoid mutants. The root is swapped back and forth
  around every pose evaluation, and forgetting to swap it back leaves the animation system
  posing a subtree.

## `CCharacterPhysicsSupport` — construction

**Contract** — allocates the character controller and configures it by creature type:

```text
actor   : a player-shaped character, actor movement restrictions, player-movable
stalker : an AI-shaped character, stalker restrictions, NOT movable by the player
biting  : an AI-shaped character, medium-monster restrictions
```

**Notes** — "movable by the player" decides whether walking into this creature pushes it.
A stalker is immovable so the player cannot shove a companion off a ledge; the default can
be overridden per spawn.

## `in_Load`

**Contract** — reads the shell control parameters and the collision-damage bounce factor,
falling back to a global collision-damage section when the creature's own section does not
name one. Note the key read from the creature's section and the key read from the global
are *different keys*, so a creature declaring the former still gets the global value — a
live defect, preserved.

## `in_NetSpawn`

**Contract** — brings the physical side up, and the order matters:

```text
FUNCTION spawn(record)
  clear the saved hit
  initialize the destructible substrate         # this clears callbacks, so it goes first
  load the death-animation table for this creature's section

  # force a pose before anything measures the skeleton
  IF dead already THEN play a corpse idle cycle
  ELSE IF no animation controls the transform THEN play a corpse idle cycle
  invalidate and recompute the bones

  spawn the skeleton substrate
  enable the character, place it at the entity's position, zero its velocity

  IF not the player THEN bounce damage factor = 1
  IF stalker THEN install the hit-flinch animation set
  record whether an animation currently controls the transform
  IF the spawn record overrides player-movability THEN apply it
```

**Invariants** — the pose must be forced before the skeleton is measured, because some
creatures are spawned with no animation assigned at all and an unposed skeleton has every
bone at the origin. The original's own comment admits it does not know why this is needed
for the *living* case; it is needed because the box that the collision correction expands
is derived from the pose.

## `SpawnInitPhysics`

**Contract** — the fork at spawn: a living creature gets a character controller (and, for
humanoids or any model whose data declares it, an inverse-kinematics foot-placement
controller); a creature spawned dead goes straight to a ragdoll. Afterwards the record's
saved per-bone physics state is validated against the actual bone count and discarded if
they disagree — a model that changed since the save cannot have its ragdoll pose restored.

**Notes** — a per-creature configuration flag can suppress character creation at spawn
entirely, named in the original as a terrible hack. It exists for creatures whose authored
spawn position is inside geometry and which must be allowed to settle before they collide.

## `CollisionCorrectObjPos` — placing without intersecting

**Contract** — the shared "put this thing here without it being inside something" routine.
Derives a box from whichever representation is about to exist, expands an activation shape
of twice that size at the target position, lets the physics layer push it out of any
overlap, and moves the entity to the result.

```text
FUNCTION correct_position(target, creating_a_character) -> did it fit
  IF creating a character THEN box = the character capsule's box
  ELSE IF a shell exists  THEN box = the shell's current bounds
                               AND temporarily disable its collision
  ELSE                         box = the entity's visual bounds

  centre, extent = box
  activation_position = centre offset by the entity's position
  result = expand an activation shape of 2 * extent at activation_position,
           pushing it out of whatever it overlaps
           (ignoring other characters when this is a corpse, and
            taking the rotation into account when this is a corpse)
  entity.position = result, minus the same offset
  re-enable the shell's collision
  RETURN whether it fit
```

**Invariants** — the shell's own collision must be disabled during its own correction, or
it pushes itself out of itself. The doubled extent gives the correction room to work; a
tight box frequently finds no free position at all.

## `KillHit` — the death transition

**Contract** — the full transition, driven by the blow that killed the creature. It builds
the shell, chooses a death animation from the blow, and then takes one of two paths.

```text
FUNCTION kill_hit(blow)
  decide whether the creature died wounded (affects friction and joint stiffness)
  remember the pre-death transform and position

  create_shell(blow.attacker) -> death_position, velocity

  animation, hit_angle = death_anims.choose(creature, blow)
  # a stalker that was already wounded, or in cover, gets no death animation:
  # its pose is already special and an animation would snap it
  IF the creature is a wounded or covering stalker THEN animation = none

  IF animation exists THEN
    replace any running interactive motion with one driven by that animation
    set it up against the shell at the blow's angle
    play it
  ELSE
    destroy the foot-placement controller

  record the blow as the fatal impulse

  IF no animation is driving THEN
    # free ragdoll: place it, fly it back, give it the character's velocity
    end_activate_free_shell(attacker, pre_death_position, death_position, velocity)
    block further hits                    # see in_Hit
```

**Invariants** — the creature's velocity at the moment of death is read *out of the
character controller* before the character is destroyed, and applied to the ragdoll. A
creature shot while sprinting must keep sprinting as a corpse.

## `CreateShell` — building the ragdoll

**Contract** — the most order-dependent routine in the file. It has to end an animation that
may be moving the creature, re-root the skeleton, evaluate the pose twice, hand ownership
of the prebuilt skeleton over to the active shell, and graft the weapon on.

```text
FUNCTION create_shell(attacker) -> death_position, velocity
  discard the collision-activation delay, any interactive animation,
          and the temporary animated collision

  IF an animation currently controls the transform THEN
    capture the animation's start transform
    destroy the animation controller
    mark the root bone's callback as overwriting     # so the animation cannot move the root

  anim_root = the skeleton's root bone
  IF humanoid THEN re-root the skeleton at the pelvis

  IF no skeleton is held ready THEN build one from the current visual

  IF this is the player and they are in a vehicle, or already removed THEN RETURN

  # evaluate the pose with the ANIMATION root, so the bounding box is right
  restore the animation root
  clear every bone's callback
  re-assert the root overwrite if an animation was running
  invalidate and recompute the bones
  re-root at the pelvis again

  IF a shell already exists THEN RETURN          # idempotent

  velocity       = the character controller's current velocity
  death_position = the controller's death position, or the entity position if none
  destroy the character controller

  # hand the held-ready skeleton over; the shell now owns it
  shell = skeleton; skeleton = none
  bind the shell to the skeleton, start simulating, adopt the entity's transform,
       install the collision callbacks
  re-assert the root overwrite
  IF this creature died wounded THEN disable its character collision

  restore the animation root; recompute the bones; re-root at the pelvis

  state = dead; skeleton_in_shell = true; death_anim_on = false

  IF single player THEN
    ask for exact integration              # ragdolls are worth the cost here
    and drop character collision once the body settles
  ELSE
    ignore collisions with dynamic bodies  # multiplayer cannot afford them
  ignore collisions with small objects
  graft the held weapon into the shell
```

**Notes**

- The pose is evaluated *twice*, both times with the animation root restored and then
  re-rooted at the pelvis afterwards. The first evaluation exists to get a correct bounding
  box; the second to give the built shell a correct starting pose. The re-rooting is what
  makes the pelvis the physical root while keeping the animation system's own hierarchy
  intact, and every recomputation has to be bracketed by it.
- Ignoring small objects and, in multiplayer, all dynamic bodies, is a straight cost
  decision: a corpse interacting with every loose can on the floor is not worth what it
  costs.

## `EndActivateFreeShell` — placing and launching the body

**Contract** — finishes a free-ragdoll death. Corrects the body out of any intersection,
then *drives it back* toward where the creature actually was, then gives it the character's
velocity scaled by who killed it.

```text
FUNCTION end_activate(attacker, original_position, death_position, velocity)
  correct the position from death_position        # may push the body some distance away
  set the shell's global transform from the corrected entity transform
  fly_to(original_position - corrected position)  # drive it back
  v = shell_control.scale_start_velocity(attacker, velocity)
  shell.linear_velocity = v
  read the transform back out of the shell
  recompute the bones
```

**Notes** — the killer influences the launch velocity: a shotgun blast throws a body, a
knife does not. That scaling lives in the shell control, not here.

## `FlyTo` — stepping the solver by hand

**Contract** — moves the ragdoll a given displacement by simulating it there, rather than
teleporting it. Freezes the physics world, disables gravity on the shell, installs a
static-environment-only contact callback, then sets a velocity that covers a tenth of the
displacement per step and steps the world ten times. Restores everything afterwards.

```text
FUNCTION fly_to(displacement)
  IF displacement is negligible THEN RETURN
  freeze the physics world
  save and clear the shell's gravity flag
  install a contact callback that only sees static geometry
  unfreeze this shell alone
  velocity = displacement / 10 / fixed_timestep
  REPEAT 10 times
    shell.linear_velocity = velocity
    step the physics world
  restore gravity, the callback data and the callback
  unfreeze the world
```

**Invariants** — this is the only place in the game layer that steps the solver directly,
and it is why the seam's stepping model must separate "step this island" from "step the
world". The static-only callback is what makes the move ignore other bodies while still
refusing to pass through walls: the body *slides* back into place and stops if a wall is
in the way.

**Notes** — ten steps is a compromise between the move being resolved in one frame and the
body tunnelling. It is a fixed count, not derived from the distance, so a long fly-back
moves faster per step and is more likely to tunnel.

## `in_Hit`

**Contract** — the damage path's physical half, and the ordering here is a sequence of
special cases each of which is load-bearing.

```text
FUNCTION hit(blow, is_killing)
  remember the blow, valid for the next second       # see in_Die
  IF removed THEN RETURN

  IF hits are blocked THEN
    # blocked for 2 seconds after a free-ragdoll death, so that the same
    # explosion does not both kill and then blast the fresh corpse
    IF 2 seconds have passed since death THEN unblock ELSE RETURN

  IF alive AND killing AND the blow is an explosion above 70 damage THEN
    dismember                                        # gib rather than ragdoll

  IF dead OR killing THEN record the blow as the kill hit
  IF no shell exists AND killing THEN kill_hit(blow)  # the death transition

  IF hit animations are enabled AND not a mutant AND no death animation is running THEN
    play a flinch animation from the blow's direction and bone

  IF no active shell THEN
    IF not killing and still alive THEN push the character controller by the impulse
  ELSE
    apply the impulse to the shell at the struck bone
```

**Invariants**

- the hit block after a free-ragdoll death exists because a killing explosion delivers its
  blow to the creature and then, on a later frame, to the corpse. Without the block the
  corpse is launched twice.
- the explosion damage threshold of 70 is the dismemberment gate. Below it a creature
  ragdolls; above it, it comes apart.

## `in_Die`

**Contract** — death arriving without a blow attached. If a hit was recorded within the last
second, re-runs it as the killing hit so the ragdoll gets the right direction and impulse.
Otherwise builds a plain ragdoll with no launch velocity and destroys the character.

**Notes** — the one-second validity window is what connects "the damage system says this
creature is dead" to "this is the blow that did it", across the frames between them.

## `in_UpdateCL`

**Contract** — the per-frame physical update, and it branches on which representation
exists.

```text
FUNCTION frame_update()
  IF removed THEN RETURN
  update the temporary animated collision, and expire it if stale
  advance the shell control's clock

  IF a shell exists THEN
    tag the shell as a ragdoll for collision filtering
    IF no death animation is driving THEN read the interpolated transform out of the shell
    ELSE                                  advance the death animation
    ensure the corpse idle cycle is playing once the death animation is done
    update friction and joint resistance    # a corpse stiffens as it settles
  ELSE IF a foot-placement controller exists THEN
    update interactive animations
    update foot placement
```

**Invariants** — the transform flows *from* the shell while dead and *to* the character
while alive. That direction reversal is the whole difference between the two
representations, and everything else in the file exists to switch it safely.

## `UpdateDeathAnims`

**Contract** — once the ragdoll is fully active and nothing is driving it, destroys the
foot-placement controller and starts the corpse idle cycle, exactly once. The animation is
started on a dead body so that its bones have *some* animated pose underneath the physics,
which the shell blends against.

## `AddActiveWeaponCollision` / `RemoveActiveWeaponCollision` — the weapon graft

**Contract** — stalkers only. On death, the held weapon's collision geometry is *moved* onto
the corpse's hand bone, so that the weapon is part of the corpse rather than a separate
body inside it. The transfer is geometric, not a joint: the geometry is detached from the
weapon's own shell and attached to the corpse's bone element, and the weapon's own shell is
then destroyed.

```text
FUNCTION graft_weapon()
  IF not a stalker, or nothing held THEN RETURN
  find the weapon bones (left hand, right hand, secondary)
  build a throwaway shell for the weapon
  attach_bone = the corpse shell element that owns the right-hand bone

  # freeze every animated bone between each hand and its physical parent,
  # so the animation cannot move a bone that now carries physical geometry
  freeze the bone chain from each hand bone up to its physical parent

  move every geometry of the weapon's root element onto attach_bone,
       re-tagging each with the attach bone's identifier
  remember them, and who owned them
  destroy the throwaway shell

FUNCTION ungraft()
  # the weapon is becoming a body of its own again: hand it back
  # its current world pose and the velocity of the point it was attached at
  compute the weapon root's transform from the grafted geometry's current pose
  set the weapon's root element to it
  remove and destroy every grafted geometry
  give the weapon root the linear velocity of that point on the corpse,
      and the corpse bone's angular velocity
  release the frozen bone chains
```

**Invariants** — the ungraft must transfer *velocity*, not only position, or a weapon
separating from a moving corpse stops dead in the air. The linear velocity is taken at the
weapon's centre of mass, which is the right point precisely because the weapon was rotating
with the bone.

**Notes** — the bone-chain freeze is why a corpse's arm does not twitch: the animation still
runs, but the bones carrying weapon geometry are pinned.

## `in_ChangeVisual`

**Contract** — the model changed underneath a creature. Rebuilds the foot-placement
controller, discards every animation-derived construct, re-reads the death and hit
animation tables against the new skeleton, and — if a ragdoll was active — tears the whole
shell down and rebuilds it from the new model.

**Invariants** — the original asserts that a stalker never reaches the rebuild path. A
humanoid corpse changing model mid-ragdoll is not supported.

## `in_NetDestroy` / `SetRemoved` / `CanRemoveObject`

**Contract** — teardown. Destroys the interactive motion, the character, both shells, the
interactive animation, the animated collision, the foot-placement controller and the
activation delay, and resets the state to alive so a recycled object starts clean.
`SetRemoved` is the lighter path used when a corpse is being taken out of the world: it
deactivates whichever shell is live and stops per-frame processing. The player may never be
removed; anything else may be removed once its sounds have finished.

## `create_animation_collision` / `update_animation_collision` / `destroy_animation_collision`

**Contract** — a *temporary* shell built around an animated (living) creature so that its
animated bones collide with the world for the duration of an animation that needs it — a
takedown, a mounted action. It is created with a three-second self-destruct that is pushed
forward every time creation is requested again, so it survives as long as something keeps
asking and goes away on its own otherwise.

## `run_interactive` / `update_interactive_anims`

**Contract** — stalkers only. When the animation system reports that the currently playing
global animation wants collision callbacks, an interactive animation is created to service
it, and destroyed when it reports it is finished.

## `on_create_anim_mov_ctrl` / `on_destroy_anim_mov_ctrl`

**Contract** — while an animation is moving the creature, the character controller is put
into a non-interactive mode: it still exists and still tracks position, but it no longer
pushes or is pushed. The original once destroyed and recreated the character instead; that
code is commented out.

## `ForceTransform` / `set_movement_position`

**Contract** — teleporting a living creature. Writes the transform, re-enables the character,
collision-corrects the new position, and zeroes the velocity. Dead creatures are ignored
entirely.

## `PHGetSyncItemsNumber` / `PHGetSyncItem`

**Contract** — what the network synchronizes: while a character exists, exactly one item —
the character controller's own state. Once it is a ragdoll, the entity's full per-bone
shell state. So a live creature costs one synchronized body and a corpse costs as many as
it has bones.

## `PHGetLinearVell`

**Contract** — the creature's velocity, from whichever representation is live.

## `CreateSkeleton`

**Contract** — builds a shell from the creature's visual: one element per physical bone,
joints from the model's own data, disable parameters (when a body may go to sleep) from the
model's embedded configuration, and an inertia smoothing pass. The smoothing factor is a
compiled-in 0.3 and blends each element's inertia tensor toward its neighbours', which
stops thin limbs from behaving wildly relative to the torso.

## `in_shedule_Update`

**Contract** — the scheduled tick: advance the collision-activation delay if one is pending,
tick the destructible substrate, and tick the character controller.

## `bone_chain_disable` / `bone_fix_clear`

**Contract** — pin a run of animated bones from a leaf up to a named ancestor, by installing
a fixing callback on each that has not got one, and release them all later. Used only by
the weapon graft.

## `DoCharacterShellCollide`

**Contract** — should this corpse collide with living characters? Yes for everything except
a stalker who died *wounded* — a body lying in the wounded pose has a shape that traps
anyone who walks into it.

## `in_NetRelcase`

**Contract** — an object is being destroyed: tell the character controller, and forget the
saved hit if that object was its attacker. The conformance invariant about dangling
references; the saved hit holds an attacker pointer across up to a second of frames.

## Could not recover

- The bounce-damage factor reads a differently-named key from the creature's section than
  the one whose existence it tests, so a per-creature factor is never actually used.
- A velocity-driven variant of the death animation exists (`imotion_velocity`) and is
  selected by a condition written `false &&`; only the position-driven variant runs.
- A routine to re-seat the physical root bone against the animation root's bind pose is
  fully written out in comments and not called. Its absence is presumably why the pelvis
  re-rooting dance exists at all.
- `in_Init` is empty. `is_similar` is defined and never used, and computes something its
  name and its parameter suggest it should not (it ignores its tolerance parameter
  entirely).
- The collision-activation delay is constructed nowhere in the live code — only in a
  commented-out branch of the spawn path — yet it is updated, destroyed and deleted twice
  in the destructor.
