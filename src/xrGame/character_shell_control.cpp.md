# src/xrGame/character_shell_control.cpp

> Tunes the ragdoll a character becomes when it dies: how hard the killing hit throws it, how fast it stiffens, and how much it slides.

**Needs** — [`character_shell_control.h`](character_shell_control.h.md) · [`Level.h`](Level.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/ExtendedGeom.h`](../xrPhysics/ExtendedGeom.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`character_shell_control.h`](character_shell_control.h.md)
**Tier floor** — T1: it writes contact parameters from inside the solver's collision callback, on the solver's own contact record

## Purpose

When a character dies its skeleton is handed to the rigid-body solver as a ragdoll. Left
alone, a ragdoll is wrong in two specific ways: it is limp for its whole fall, where a real
body's joints resist; and it slides on whatever it lands on, where a body comes to rest.

This object is the correction. It ramps joint resistance up from nothing to a configured
maximum over a configured interval measured from the moment of death, and it ramps contact
friction from one configured value to another over a second interval, feeding the friction
into the solver per contact. The body therefore falls loosely, stiffens as it settles, and
grips where it lands.

It also owns two smaller decisions about the killing hit itself, and one detection: whether
the character was *already down* when killed, which changes the settling profile entirely —
a body that falls from standing needs a long settle; a body that was already lying wounded
must not slide off across the floor when finished off.

## State

```text
RECORD character_shell_control
  # ragdoll body properties, from the creature's configuration section
  air_resistance_linear, air_resistance_angular : real
  joint_resistance_max        : real       # the stiffness reached at the end of the ramp
  joint_ramp_duration         : real       # seconds from death to full stiffness
  joint_ramp_remaining        : real       # counts down; 0 == fully stiff

  # contact friction ramp -- two profiles, standing and already-wounded
  friction_start, friction_end : real
  friction_ramp_duration            : real
  friction_ramp_remaining           : real
  friction_ramp_duration_wounded    : real
  friction_ramp_remaining_wounded   : real
  current_friction            : real       # what the contact callback writes

  # killing-hit shaping
  fatal_impulse_factor        : real
  shot_up_factor              : real       # 0 == no upward bias
  after_death_velocity_factor : real       # defaults to 1

  # wounded detection
  has_wounded_state           : bool       # whether this creature type can be wounded at all
  pelvis_ground_probe         : real       # probe length below the pelvis
  was_wounded                 : bool

  previous_time, time_delta   : real       # global clock, for the ramps
```

**Invariants**

- Both ramps are driven by a *wall-clock delta* the object computes itself, not by the
  physics step or the frame delta. The ragdoll is updated at whatever rate the object is
  scheduled at, which degrades with distance, so a fixed per-update decrement would make
  distant corpses stiffen at a different rate from near ones.
- The first delta after a reset is zero, not the time since the epoch. Without that a body
  would stiffen instantly on its first update.
- `was_wounded` is decided **once**, at death, and selects which of the two friction
  profiles is used for the whole settle. Re-evaluating it as the body falls would switch
  profiles mid-fall.
- `current_friction` is read from inside the solver's contact callback and written from the
  update. They do not run concurrently, but the update must have run at least once before
  the first contact or the friction read is uninitialized.

## configuration

**Contract** — ten values in the creature's section, all named `ph_*`: the two air-resistance
factors, the joint stiffness maximum and its ramp duration, the fatal-impulse factor, the
friction ramp's duration, start and end, whether the creature has a wounded state and the
wounded ramp duration, and the pelvis probe length. Two further values — the upward bias on
the killing shot and the post-death velocity factor — are **optional**, defaulting to no
bias and no scaling, so a creature type that does not declare them is unaffected.

**Invariants** — each ramp's remaining time is initialized to its full duration at load, so
"loaded" and "just died" are the same state. A rebuild that separates loading from death
must reset the remainders at death.

## `set_kill_hit`

**Contract** — biases the direction of a killing hit upward by a configured amount and
renormalizes, **except for explosions**. Bodies that simply crumple where they stood read as
limp; a small upward component makes the death legible. Explosions are excluded because they
already impart their own, much larger, directional impulse and adding to it launches the
body.

## `set_fatal_impulse`

**Contract** — multiplies the killing hit's impulse by the fatal-impulse factor, **unless the
character was already wounded**, and not for explosions (whose factor is one). A finishing
shot on someone already lying down should not throw the body across the room; that is the
single most visible ragdoll artefact the system exists to avoid.

## `apply_start_velocity_factor`

**Contract** — scales the velocity the ragdoll is launched with, by a fixed factor and the
configured post-death factor — and applies the configured factor **a second time when the
killer is not an anomaly**.

```text
FUNCTION apply_start_velocity_factor(killer, velocity)
  velocity = velocity * 1.3 * (1.25 * after_death_velocity_factor)
  IF killer is not a zone THEN
    velocity = velocity * (1.25 * after_death_velocity_factor)
```

**Notes** — anomalies get one application rather than two because an anomaly has already
applied its own force to the body over the seconds before it killed it; doubling on top of
that throws corpses out of the anomaly.

**Notes** — the two bare constants, 1.3 and 1.25, are not configurable and not derived. They
are a global "make deaths more dramatic" multiplier applied in two places for no discoverable
reason; a rebuild should collapse them into one factor.

## `set_start_shell_params`

**Contract** — applied once when the ragdoll is created: sets its linear and angular air
resistance from configuration, installs the contact callback below, and gives the solver a
back-reference to this object so the callback can find it.

## the contact callback

**Contract** — invoked by the solver for each contact involving the ragdoll, before the
contact is solved. Overwrites the contact's friction coefficient with the current ramped
value, discarding whatever the two materials would have produced.

**Invariants** — material-derived friction is *replaced*, not scaled. A corpse's slide is
decided entirely by where it is in its settle ramp, not by what it landed on. That is a
deliberate simplification: the ramp's whole job is to bring the body to rest in a bounded
time regardless of surface.

**Notes** — the callback recovers this object from user data hanging off whichever of the two
contacting geometries is the ragdoll's, which is the solver's only channel for per-object
context. A rebuild whose contact callback can carry a typed context needs none of it.

## `TestForWounded`

**Contract** — decides, once at death, whether the character was already on the ground.
Forces a bone evaluation, takes the pelvis bone's world position, and probes straight down
against the **static** collision database for the configured distance. Anything found means
the pelvis was close to the floor, so the character was lying wounded rather than standing.
Creature types without a wounded state skip the test and are never wounded.

```text
FUNCTION test_for_wounded(creature_transform, model)
  was_wounded = false
  IF NOT has_wounded_state THEN RETURN
  model.evaluate_bones()
  pelvis = creature_transform COMPOSED WITH model.bone_transform("bip01_pelvis")
  IF a downward ray from pelvis.position hits static geometry within pelvis_ground_probe THEN
    was_wounded = true
```

**Notes** — the probe is against static geometry only, so a character lying on a table or a
vehicle is judged standing. In practice wounded characters lie on the floor.

**Notes** — the pelvis bone is named literally. The entire hit-reaction and death system
assumes the shipped biped skeleton's bone names; a rebuild loading the shipped models
inherits those names and cannot rename them.

## `CalculateTimeDelta`

**Contract** — computes the elapsed wall-clock time since the previous call, yielding zero on
the first. Split from the ramp update because several ramps share one delta and the delta
must be taken once per update, not once per ramp.

## `UpdateFrictionAndJointResistanse`

**Contract** — advances all three countdowns by the delta, clamping at zero, then computes and
applies the two ramped quantities. Joint resistance rises linearly from zero to its maximum;
friction falls linearly from its start value to its end value. Which friction profile is used
is decided by the wounded flag.

```text
FUNCTION update_ramps(shell)
  decrement joint_ramp_remaining, friction_ramp_remaining,
            friction_ramp_remaining_wounded by time_delta, each clamped at 0

  shell.joint_resistance = joint_resistance_max
                           * (1 - joint_ramp_remaining / joint_ramp_duration)

  IF was_wounded THEN (duration, remaining) = wounded profile
  ELSE                (duration, remaining) = standing profile

  current_friction = friction_end
                     + (remaining / duration) * (friction_start - friction_end)
```

**Invariants** — all three countdowns are decremented every update even though only one
friction profile is read, so that the unused one does not sit full. Nothing switches profiles
mid-settle, so this costs nothing and guards against a future that does.

**Notes** — the durations are read as seconds and used as seconds, but the source comment
describes converting them from *frames*. The conversion does not exist; the comment is stale.
A rebuild should treat every one of these as seconds, which is what the shipped data means.
