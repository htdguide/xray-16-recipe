# src/xrGame/ActorAnimation.cpp

> Chooses, every frame, which three motions the actor's body plays — legs, torso and head — and bends the spine so the body follows where the camera is looking.

**Needs** — [`Actor.h`](Actor.h.md) · [`ActorAnimation.h`](ActorAnimation.h.md) · [`actor_anim_defs.h`](actor_anim_defs.h.md) · [`Weapon.h`](Weapon.h.md) · [`Missile.h`](Missile.h.md) · [`Artefact.h`](Artefact.h.md) · [`Inventory.h`](Inventory.h.md) · [`Level.h`](Level.h.md) · [`Car.h`](Car.h.md) · [`player_hud.h`](player_hud.h.md) · [`IKLimbsController.h`](IKLimbsController.h.md) · [`step_manager.h`](step_manager.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Motion.hpp`](../xrCore/Animation/Motion.hpp.md)
**Used by** — [`ActorAnimation.h`](ActorAnimation.h.md)
**Tier floor** — T2: per-frame matrix composition on the skeleton's evaluation path

## Purpose

The actor's third-person body is driven by data, not by a state machine: every pose the
actor can be in corresponds to a motion whose *name* is built by string concatenation from
the posture, the equipped item's animation slot and the item's current state. This file is
the two halves of that: the **name-building tables**, filled once when the actor's visual
is bound, and the **per-frame selector** that walks a decision tree over the movement
bitset and the active item's state to pick one motion for each of the three body
partitions.

The second thing it owns is the **aim spine bend**: the actor looks with the camera, but
the camera can pitch and yaw far beyond what any authored motion covers, so four bones —
two spine bones, the shoulders and the head — are rotated after the pose is evaluated, by
fractions of the camera's offset from the body's facing. This is why the third-person
actor can be seen aiming straight up while playing a flat idle.

## State

All of it is motion identifiers resolved once, at visual-bind time, from names. Nothing
here is per-frame state except the three "what is currently playing" fields on the actor.

```text
RECORD TorsoWeaponSet          # one per animation slot, per posture
  moving[idle|walk|run|sprint] : motion   # "<posture>_torso<slot>_aim_1..3", sprint uses "_escape_0"
  zoom, holster, draw          : motion
  reload, reload_1, reload_2   : motion   # begin / in-process / end, for tri-state reloads
  drop, attack, attack_zoom    : motion
  fire_idle, fire_end          : motion
  all_attack_0..2              : motion   # whole-body variants, used only while standing still

RECORD LegSet                  # one per gait
  fwd, back, left_strafe, right_strafe : motion   # "<posture>_<gait>_<dir>_0"

RECORD PostureSet              # one per posture: normal, crouch, climb
  legs_idle, legs_turn, death          : motion
  walk, run                            : LegSet
  torso[13]                            : TorsoWeaponSet    # indexed by animation slot
  torso_idle, head_idle                : motion
  jump_begin, jump_idle, landing[2]    : motion
  damage[12]                           : motion            # additive hit reactions, by direction
```

**Invariant** — the torso table has exactly thirteen entries because the game defines
thirteen animation slots, and an item's slot indexes it directly (one-based in the data,
so the lookup subtracts one). Adding an item class that needs a new hold pose means adding
a slot *and* thirteen new motion names to every posture's data. That coupling between a
C++-side constant and a naming convention in the animation bank is the most brittle thing
in this file.

**Invariant** — motions are resolved in two strengths. The gait and idle motions are
resolved *strictly*: a missing name is a fatal data error, because there is no sensible
fallback for "how does the actor stand". The weapon-hold motions are resolved *leniently*
and may come back invalid, because not every slot has every state — a knife has no
tri-state reload — and the selector falls back to the idle hold.

### Bend factors

```text
                 yaw    pitch  roll
  lower spine    0.00   0.00   0.30
  upper spine    0.40   0.20   0.30
  shoulders      0.40   0.70   0.20
  head           0.20   0.10   0.20
```

They sum to 1.0 in yaw and pitch, which is the whole design: the *total* rotation applied
across the chain equals the camera's offset exactly, so the gun ends up pointing where the
player is aiming, while the distribution across the chain decides whether the actor looks
like it is turning its head or twisting its whole body. Roll deliberately sums to 1.05 and
is distributed toward the spine, because a leaning actor should lean from the waist.

## bone bend callbacks — spine, shoulders, head

**Contract** — each is installed on one bone and runs during skeleton evaluation, after
the animation has produced that bone's transform and before its children are evaluated.
Each rotates the bone in place by its share of the camera's offset from the model's
facing, preserving the bone's translation exactly so the skeleton's proportions do not
change.

```text
FUNCTION bend_bone(bone, yaw_share, pitch_share, roll_share)
  # The offset is how far the aim has turned away from where the BODY faces;
  # the body's own turning (model yaw plus the in-progress turn delta) is
  # already in the animation, and must not be counted twice.
  yaw   = normalize_signed(aim.yaw - model_yaw - model_yaw_delta) * yaw_share
  pitch = normalize_signed(aim.pitch) * pitch_share
  roll  = normalize_signed(aim.roll)  * roll_share

  origin = bone.transform.translation
  bone.transform = bone.transform composed with rotation(-pitch, yaw, roll)
  bone.transform.translation = origin        # rotate about the joint, never displace it
```

**Notes** — the pitch is negated because the skeleton's convention has pitch increasing
downward while the camera's increases upward. The vehicle variant uses a different
rotation-order convention and fixed three-quarter shares on yaw and pitch, and does *not*
subtract the model yaw: a seated actor's body does not turn, so the head takes the whole
offset, clamped implicitly by the seat's own aim limits.

## motion-table construction

**Contract** — called once when the actor's visual is bound. Builds every table above by
concatenating a posture prefix, a fixed infix and a slot or index suffix, and asking the
animated visual to resolve each name. Three postures are built — `norm`, `cr` and a climb
posture — plus a sprint set and a per-vehicle-type steering set.

**Notes**

- The **climb posture is a hybrid** and this is load-bearing: its idle, torso-idle and
  both gaits come from the climb bank, but its turn, death, all thirteen weapon-hold
  tables, its jump and landing motions and its hit reactions are taken from the *normal*
  bank. A ladder has no authored weapon poses, so the standing ones are reused. Its head
  idle is deliberately left invalid, which suppresses head animation entirely while
  climbing.
- Both the walk and run gaits of the climb posture resolve to the same `_run` names. There
  is one ladder speed.
- The **sprint set is separate** from the posture sets because sprinting has no torso
  variation at all — the actor cannot aim while sprinting — and has jump variants the
  other gaits lack. Those jump variants are resolved leniently and fall back to the
  grounded sprint motion when absent.
- The **vehicle steering set** is built per driver-animation-type: a left-steer, a
  right-steer and up to a fixed number of idle variants discovered by probing numbered
  names until one is missing. Probing rather than declaring the count means the data can
  add idles without a code change, and it means a *gap* in the numbering silently
  truncates the set.

## `g_SetAnimation`

**Contract** — the per-frame selector. Given the actor's movement bitset, picks a motion
for legs, torso and head, and starts any that differs from what is currently playing.
Starting a motion the actor is already playing is skipped, which is what keeps a
continuous walk cycle continuous.

**Invariants** — a dead actor plays nothing: the current-motion fields are cleared and the
movement state is zeroed, so the ragdoll is not fighting an animation. The three
partitions are always assigned something; every branch that can leave a slot empty is
followed by a fallback.

```text
FUNCTION set_animation(movement_bits)
  IF NOT alive THEN clear current legs/torso/head; movement_state = 0; RETURN

  posture = crouch bit ? crouch : climb bit ? climb : normal
  fast    = is_accelerated(movement_bits, aiming)        # the one shared predicate
  gait    = fast ? posture.run : posture.walk
  moving  = not moving ? idle : fast ? run : walk        # index into the torso hold table

  legs  = select_legs(movement_bits, posture, gait)      # priority order below
  IF sprinting THEN legs = sprint set by direction; moving = sprint
  IF this is the viewed entity AND the sprint or moving bits just changed THEN
    tell the first-person weapon model the movement changed

  IF climbing THEN torso = the same directional leg motion   # both halves climb together
  ELSE torso = select_torso_from_active_item(posture, moving)

  fill any empty partition from posture idles
  start whichever of the three changed
```

### leg selection priority

The order is a priority list, not a switch, and the order is the decision:

```text
landing (hard)  >  landing (soft)  >  turning in place (never while climbing)
 >  falling  >  jump start  >  forward  >  backward  >  left strafe  >  right strafe
 >  standing still
```

Landing beats everything because a landing that is interrupted by a movement input reads
as the actor never having landed. Turning-in-place is suppressed while climbing because a
ladder has no turn. "Standing still" is recorded as a flag rather than as a motion,
because several torso cases below need to know it.

**Phase preservation** — when the leg motion changes *while already moving*, the new
motion starts at the same normalized phase the old one was at. Without this, changing
direction mid-stride resets the gait cycle and the actor visibly hitches. When movement
*starts* from a standstill, the phase is forced to the middle of the cycle, so the first
step is taken with the other foot than the last time — the alternative, always starting at
zero, makes every departure identical.

**Notes** — the leg motion is started "by parts", meaning the same motion is applied to
several bone groups independently rather than as one blend; the completion callback
invalidates the current-leg field so the next frame re-selects. The step manager is
notified of the new leg blend so it can schedule footstep sounds against the motion's own
event markers rather than against a timer.

### torso selection

**Contract** — the torso is whatever the *active inventory item* says it is. The item's
animation slot picks a hold table; the item's own state machine picks an entry within it.
Four item families are distinguished and each has its own mapping:

- **a knife** (identified by which inventory slot is active, not by class): its two fire
  states play whole-body attack motions when the actor is standing still, and torso-only
  ones when moving, because a lunge that moves the legs cannot be played while the legs
  are walking;
- **any other weapon**: idle, fire and the fire-alternate each have a zoomed and an
  unzoomed variant; a reload is either one motion or the three-part begin/in-process/end
  sequence depending on whether the weapon declares a tri-state reload;
- **a thrown item**: five states — showing, ready, throw start, throw, throw end — again
  in a standing whole-body form and a moving torso-only form;
- **an artefact being activated**: the activation reuses the *zoom* hold, because the
  gesture is the same.

**Priority above all of these**: if a drop is pending — the actor has released the throw
button but the drop animation has not been triggered — the drop motion wins outright and
latches a flag that suppresses further torso selection until the drop's own callback
clears it. A missing drop motion logs and falls back to the torso idle rather than
freezing.

**Notes** — the check that the item's animation slot is within the table is an assertion,
not a clamp. Data that declares a slot beyond the thirteen is a content bug the engine
refuses to paper over.

### torso/leg synchronisation

**Contract** — the final act of the selector. When *both* the torso and the leg motions
declare themselves as synchronised parts, the torso's playback time is forced to the leg
motion's normalized phase. This is how a walk cycle's arm swing stays in step with its
footfalls when the two halves come from different clips at different lengths.

**Invariants** — the synchronisation is one-directional: legs are the master. Both motions
must opt in through a flag in their own definition data, so a clip that was not authored
to sync is left alone.

## `steer_Vehicle`

**Contract** — plays a steering pose on the actor's own skeleton while driving. Three
cases only: neutral plays the first idle of the vehicle's driver-animation type, and a
non-zero steering angle plays the full left or full right pose. There is no blending by
angle; the transition between the three poses is the animation system's own cross-fade.

**Notes** — the extra idle variants the table discovers are never selected here. Only
index zero is ever played, so the probing loop's result beyond the first entry is unused
in the shipped code.

## `FindMotionKeys`

**Contract** — returns the root-bone motion track of a named motion on a given visual, or
nothing when the visual is not animated or the motion is invalid. Callers use it to read
how far a motion *would* move the actor before deciding to play it.

## Notes

**Debug overlays.** A compiled-out block prints the selected motion names, the movement
bitset, the physics environment (on ground / in air / at wall), the actor's navigation
vertex, position, acceleration and three velocity readings. A rebuild should keep the
*list* — those seven readings are what someone debugging actor movement actually needs —
even if the presentation differs.

**A stopped-dead motion is resolved and never played.** The dead-stop motion is built into
the table and its only use site is commented out; the dead actor is handed to the ragdoll
instead. A rebuild should drop it.
