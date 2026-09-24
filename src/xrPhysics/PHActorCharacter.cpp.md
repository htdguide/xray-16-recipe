# src/xrPhysics/PHActorCharacter.cpp

> What makes the player's controller different from a creature's — jump legality,
> a named material that cannot be climbed, spacing from other creatures, and two different
> sets of collision rules for single-player and multiplayer.

**Needs** — [`PHActorCharacter.h`](PHActorCharacter.h.md) · [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`ElevatorState.h`](ElevatorState.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHActorCharacter.h`](PHActorCharacter.h.md)
**Tier floor** — T1: it builds solver shapes and writes contact constraints by hand inside
the collision pass.

## Purpose

Everything the player's controller does that a creature's does not. Four groups: the
restrictor cylinders and their lifecycle, jump legality, the contact rules (which fork
completely between single-player and multiplayer), and two small corrections applied while
the player is out of control.

## State

```text
RECORD ActorController EXTENDS SimpleCharacter
  restrictors  : list<Restrictor>      # exactly three: stalker, small stalker, medium monster
  single_game  : bool                  # fixed at construction, never changed
  speed_goal   : real                  # declared, unused by the shipped code

MODULE STATE
  slide_material : material index      # the "earth_slide" material, resolved once, lazily

CONSTANTS
  jump_up_velocity_default    = 6.0    # metres per second; the controller may override
  jump_carry_forward_rate     = 1.2    # how much horizontal speed a jump keeps
  jump_steer_fraction         = 0.2    # how much of the desired direction a jump adds
  climb_jump_scale            = 0.5    # a jump off a ladder gets half the strength
  free_fall_up_force_limit    = 4000   # see "out of control"
  acceleration_change_epsilon = 0.05 direction, 0.5 magnitude
```

**Invariants** — the restriction type of the player itself is `actor`, set at construction
and never changed; the three restrictors it carries represent the classes of *other*
creatures it wants spacing from, not its own.

The slide material is resolved from the material library by name on first creation and
cached in module state. Its index is used by two independent decisions, so a rebuild that
loses the lookup silently disables both — the failure is a player who can climb an
unclimbable slope, not a crash.

## restrictor lifecycle

**Contract** — three restrictors are built at construction and created with the body when
the character is created, sized to the character's height. They follow the character's
reference object and material, and are destroyed with it.

**Notes** — in multiplayer the restrictors are **discarded outright** on creation, before
they are ever built. Spacing volumes are a single-player courtesy: in a deathmatch, players
must be able to crowd each other, and an invisible cylinder holding two players a metre
apart is a bug, not a nicety.

Each restrictor is attached to the character's body through a transform node rather than
directly, because the cylinder must sit offset from the body's origin. The node owns the
placement and the cylinder owns the shape; destroying them is two steps and the transform
node must not be told to clean up its child, because the child is destroyed explicitly. That
is the incidental half; the load-bearing half is that a restrictor's *placement* is fixed
relative to the body and never animated.

## `ChooseRestrictionType` — growing and shrinking the spacing

**Contract** — called from a restrictor contact. The player's large-stalker restrictor
inspects the creature that touched it and may change that creature's size class.

```text
FUNCTION choose_restriction_type(my_type, depth, other)
  IF my_type is not the LARGE stalker class          RETURN
  IF other is not a stalker of either size           RETURN
  threshold = the radius of MY small-stalker restrictor
  IF other is currently SMALL and its own radius > threshold
      queue other to become LARGE                    # deferred: applied between steps
  ELSE IF other is currently LARGE and its own radius < threshold
      set other to SMALL immediately
```

**Invariants** — growing is deferred to the end of the step and shrinking is immediate. That
asymmetry is the file's subtlest decision: shrinking a creature's spacing can never create a
new penetration, so it is safe inside a contact callback; growing one can, and a shape that
grows while the solver is mid-collision leaves contacts describing geometry that no longer
exists. The two-field pending-change mechanism in
[`PHCharacter.h`](PHCharacter.h.md) exists for exactly this.

**Notes** — the threshold is the player's own small-stalker restrictor radius, which is set
from game data. The rule reads as: a creature is "large" if its own body radius exceeds the
spacing the player reserves for small ones. Classification is therefore relative to the
player's configuration rather than to an authored per-creature flag, which is why a mod that
changes the player's restrictor radii also changes which NPCs keep their distance.

The player is woken on either change, because a spacing change with a sleeping player would
take effect only when something else happened to wake it.

## jump legality

**Contract** — a jump is refused unless the player is in control, is not standing on the
slide material, and is either on ground that is not too steep or currently climbing.

```text
FUNCTION can_jump() -> bool
  RETURN in_control
     AND last material is not `earth_slide`
     AND (ground normal's vertical component > 0.5  OR  currently on a ladder)
```

```text
FUNCTION jump(desired_direction)
  IF NOT can_jump()   RETURN
  IF currently climbing
      direction = the ladder's own jump-off direction, given the desired direction
      jump velocity = direction * (jump_up_velocity / 2)
  ELSE
      jump velocity = ( current horizontal velocity * 1.2 + desired direction * 0.2,
                        jump_up_velocity,
                        same for the other horizontal axis )
  wake the body
```

**Invariants** — a ground normal with a vertical component above one half is a slope of at
most sixty degrees. That single number is the engine's slope limit for jumping, and it is
not the same as the slope limit for walking (which lives in the shared controller).

**Notes** — the jump is a *velocity assignment*, not an impulse, and it keeps 120% of the
horizontal speed the player already had. A running jump therefore goes further than a
standing one and slightly faster than the run itself, which is the classic first-person
bunny-hop and is here on purpose. The desired direction contributes only a fifth of a unit,
normalised — enough to steer a jump, not enough to launch from standing.

A jump off a ladder takes its direction from the climbing state (see
[`ElevatorState.cpp`](ElevatorState.cpp.md)) and half the strength, because a ladder jump is
a push away from a surface the player is already attached to, not a leap.

`earth_slide` is a material authored on scree slopes. Standing on it forbids jumping *and*
disables the climb-assist that would otherwise let the player walk up a step — see below.
It is the engine's only hard-coded material name outside the material system itself, and a
rebuild must ship a level material with that exact name or those slopes become climbable.

## contact rules, single-player

**Contract** — invoked for every contact the player's shapes generate, before the solver
builds a constraint.

```text
FUNCTION init_contact_singleplayer(contact, do_collide, material_a, material_b)
  from_restrictor = the contact involves one of my restrictor cylinders
  IF either material is flagged "actor obstacle"
      force do_collide = true              # this material stops the player regardless
  IF from_restrictor
      note that I have a side contact
      contact.friction = 0                 # spacing must never grip
  ELSE
      run the ordinary character contact rules
  IF from_restrictor AND the contact stands AND the other character is NOT actor-movable
      build a ONE-SIDED contact constraint attached to me and to nothing
      suppress the solver's own contact
      scale my friction factor by 0.1      # see Notes
```

**Invariants** — the one-sided constraint against an immovable creature is the same device
used throughout the chapter: the player is stopped by the creature, and the creature is not
pushed by the player. Actor-movable creatures get an ordinary two-sided contact and do get
shoved.

**Notes** — the "actor obstacle" material flag forces collision on even when every other
rule would have suppressed it. It is the level designer's override for invisible barriers
and for thin geometry the player must not pass — it wins over the passable flag, over the
restrictor rules, and over everything else in this function.

Scaling the player's friction by a tenth when held off by an immovable creature is what
stops the player sticking to an NPC's spacing volume. Without it, walking into a stationary
stalker and continuing to push produces a static-friction lock that reads as the player
being glued to empty air a metre from the NPC.

## contact rules, multiplayer

**Contract** — a different set. Player-versus-player contact is admitted only when both
players are in the same physical state, and every contact between fast-moving characters is
converted into two one-sided constraints.

```text
FUNCTION init_contact_multiplayer(contact, do_collide, material_a, material_b)
  IF both sides are actors
      do_collide = do_collide
                   AND the contact is not from a restrictor
                   AND both sides are EITHER both ragdolls OR both live controllers
      contact.friction = 1
  IF do_collide   run the ordinary character contact rules
  separate_if_fast(contact, do_collide)
```

```text
FUNCTION separate_if_fast(contact, do_collide)
  IF either side is not a character                 RETURN
  soften the contact: compliance * 100, error reduction * 0.1
  IF both characters are slower than 2 m/s          RETURN
  contact.friction = 1
  build TWO one-sided contact constraints from the same contact, one attached to each body
  wake both characters
  suppress the solver's own contact
```

**Invariants** — "both ragdolls or both live" is expressed as a comparison of whether each
side has a physics shell. A live player has none (it is a character controller); a dead one
has a ragdoll. Mixing the two produces a corpse that a running player can kick across the
map, which is why the case is excluded rather than tuned.

**Notes** — the two one-sided constraints are the multiplayer collision model in one line.
An ordinary shared constraint couples two players into one solver island, so one player's
latency spike perturbs the other's trajectory and the two clients diverge. Two independent
one-sided constraints let each player be stopped by the other *without the solver ever
coupling them*, which keeps each client's prediction of its own player reproducible. This is
the chapter's direct contribution to
[criterion 14](../../SYSTEM-REQUIREMENTS.md#6-conformance).

The separation is applied only above two metres per second because the constraint pair is
not physically correct — momentum is not conserved, both players get pushed as if the other
were immovable — and at walking speeds the error is visible as two players failing to nudge
each other. Above running speed nobody can tell, and the divergence it prevents matters more.

## out of control

**Contract** — two corrections applied while the player has lost control (thrown by a blast,
caught in an anomaly).

- **`ValidateWalkOn`** — if the player is standing on the slide material, the climb-assist
  that lets a walking character mount a step is switched off; otherwise the shared rule
  applies.
- **`PhTune`** — while out of control and not under a deliberate external impulse, the
  upward component of the accumulated force on the body is capped at a fixed limit.

**Notes** — the upward force cap is a ceiling on how hard the world may throw the player
skyward. It is applied only when the player is not in control *and* is not being
deliberately impulsed, so a scripted launch still works; what it catches is the pathological
case where a player wedged between two surfaces accumulates contact force until the solver
ejects them out of the level. The limit's value is an absolute force and therefore depends
on the player's mass, which is a latent coupling the source does not acknowledge.

## acceleration filtering

**Contract** — a new desired acceleration is passed down to the shared controller only when
it differs from the current one by more than a small angle or a small magnitude.

**Notes** — this is not an optimisation. Setting the desired acceleration wakes the body and
resets internal movement state; a game layer that re-sends the same intent every frame
(which it does) would keep the player permanently awake and re-latch the movement state
machine every tick. The thresholds — about three degrees of direction, half a unit of
magnitude — are the granularity below which the player's intent is treated as unchanged.

## `update_last_material`

**Contract** — the player refreshes the material underfoot only when the material currently
recorded is one flagged "actor obstacle". Otherwise the recorded material stands.

**Notes** — read the other way round, this says an invisible-barrier material is never
allowed to persist as "the surface the player is standing on". Those materials are authored
on collision-only geometry — clip brushes, blockers — and the moment one is recorded it is
replaced at the next opportunity, because the footstep sounds, the dust and the AI's
perception of the ground all read that field. Every other material is sticky: once the
player is recorded as standing on metal, only leaving the metal changes it.

The recorded material is held indirectly (see [`PHCharacter.h`](PHCharacter.h.md)), which is
what lets a vehicle or a mounted position pin the surface underfoot to its own material
without the controller knowing what a vehicle is.
