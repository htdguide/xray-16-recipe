# src/xrGame/Explosive.cpp

> What it means to explode: a blast wave that tests line of sight with sampled rays, a cloud of simulated fragments, a light, a sound, a decal and a screen shake — spread over several frames because a grenade in a crowded room cannot be resolved in one.

**Needs** — [`Explosive.h`](Explosive.h.md) · [`Entity.h`](Entity.h.md) · [`Weapon.h`](Weapon.h.md) · [`Actor.h`](Actor.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`Level.h`](Level.h.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`wallmark_manager.h`](wallmark_manager.h.md) · [`game_base_space.h`](../xrServerEntities/game_base_space.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/IActivationShape.h`](../xrPhysics/IActivationShape.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`xrPhysics/MathUtils.h`](../xrPhysics/MathUtils.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrEngine/profiler.h`](../xrEngine/profiler.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — reached through its declarations in [`Explosive.h`](Explosive.h.md); callers name that, not this file.
**Tier floor** — T2: ray queries against the collision database and a few dozen bullets injected; the cost, not the representation, is what constrains it

## Purpose

Every explosion in the game — grenades, rockets, exploding vehicles, anomalies that
detonate, barrels — is this file. It is inherited into an object rather than being a
standalone effect, because an explosion needs the exploding object's position, velocity,
bounding box and network identity.

The design answers one question well: **how much of a blast reaches a given object?** The
naive answers are both wrong. Distance alone lets a grenade kill through a wall. A single
line-of-sight ray lets a creature survive behind a lamp post. The answer used here is to
fire a handful of rays from random points inside the explosion's own volume to random
points inside the target's bounding box, weight each by how much of the target that ray's
direction actually presents, and multiply by an attenuation over distance and by how much
each material along the way stops. The result is a single factor between zero and one that
scales both the damage and the impulse.

The second design decision is that an explosion has **duration**. It is not an instant;
it is a state the object is in for a configured time, during which the light decays, the
particles follow, and the blast wave is applied to at most three objects per frame. A
grenade in a room with twenty creatures resolves over seven frames.

## State

```text
RECORD BlastParameters             # all from one configuration section
  blast_hit, blast_impulse, blast_radius  : real
  frag_count                               : int
  frag_hit, frag_impulse, frag_radius      : real
  fragment_speed                           : real
  hit_type_blast, hit_type_frag            : hit type   # named in data, not fixed
  up_throw_factor                          : real       # bias the impulse upward
  wallmark_size                            : real       # invariant: strictly positive
  explode_particles                        : text
  light_colour, light_range, light_time    : colour, real, real
  explode_duration                         : real
  hide_in_explosion                        : bool       # false only for a smoke grenade
  explode_hide_duration                    : real       # when not hidden immediately
  dynamic_particles                        : bool       # the effect follows the object
  effector_section                         : text       # the screen shake

RECORD ExplosionState
  flags        : { exploding, event_sent, ready_to_explode, exploded }
  position, direction, size  : vector
  duration_left              : real
  blasted_objects            : list<object>   # found once, drained a few per frame
  initiator                  : entity identifier
  light                      : optional<light>
  particle                   : optional<effect>    # kept only when dynamic
  already_hidden             : bool
```

Invariants:

- the four flags are a *sequence*, not independent bits: ready → exploding → event sent →
  exploded. The object is `Useful` — reusable, as a pooled item — only while all four are
  clear.
- an explosion may only be triggered once. The event that triggers it is ignored if the
  object is already exploding.
- the initiator must be a valid entity identifier before the explosion runs; it is
  asserted. An explosion with no known author is a bug, because the kill has to be
  attributed.
- **nothing here may run inside a physics step.** Asserted at every entry point: the
  explosion creates lights, plays sounds, injects bullets and disables collision, none of
  which a physics callback may do.

## `ExplosionEffect` — how much blast reaches an object

**Contract** — the central computation. Given the explosion's centre and radius and a target
object, return the fraction of the blast that reaches it, between zero and one.

```text
FUNCTION explosion_effect(explosive, target, centre, radius) -> real
  express the centre in the target's local frame
  IF the target's bounding box CONTAINS the centre THEN RETURN 1    # point blank

  volume = the box's volume
  max_section = volume / the box's smallest dimension    # the largest cross-section it has
  IF the target has an active physics assembly with a smaller volume THEN use that instead

  effect = 0
  REPEAT 5 times
    end   = a random point inside the target's box, in world space
    start = a random point inside the EXPLOSION'S OWN box
    IF start is inside the target's box THEN effect = effect + 1; CONTINUE

    direction = normalize(end - start); range = |end - start|
    # how much of the target this ray direction actually presents
    section = volume * ( |direction . target x| / size.x
                       + |direction . target y| / size.y
                       + |direction . target z| / size.z )
    effect = effect + sqrt(section / max_section) * TestPassEffect(start, direction, range, radius)
  RETURN effect / 5
```

**Invariants** — five rays is not a tuning knob that trades quality for speed in the usual
way: the count is fixed, and the *random* endpoints mean the result differs slightly between
two identical explosions. That is deliberate — a deterministic sampling would make cover
either perfect or useless at a given geometry, and the noise makes partial cover feel
partial.

The cross-section weight is the interesting half. A ray that arrives along the target's
thinnest axis presents a large face and counts for more; one that arrives along the long
axis presents a small face and counts for less. Using the physics assembly's volume when it
is smaller than the bounding box's stops a sprawling ragdoll from being treated as a
building-sized target.

**Notes** — the start point is sampled from the *explosion's* box, not from its centre,
which is what lets a large explosion partially wrap around cover. `GetRayExplosionSourcePos`
supplies it, and subclasses override it — an exploding vehicle samples a point inside the
vehicle.

### `TestPassEffect`

**Contract** — attenuate one ray by distance and by every material it passes through.

```text
FUNCTION test_pass_effect(start, direction, range, effect_radius) -> real
  r2 = effect_radius squared
  distance_factor = r2 / (range^2 * (extinction - 1) + r2)
  IF range is negligible THEN RETURN distance_factor
  cast a ray of that length against BOTH static geometry and objects, ignoring the target
  FOR EACH thing hit, in order
    look up its surface material
    shoot_factor = shoot_factor * (1 - that material's shoot-through factor)
    STOP EARLY once shoot_factor falls below 0.01
  RETURN shoot_factor * distance_factor
```

**Invariants** — the distance falloff is *not* inverse-square. It is shaped so that at
exactly the blast radius the effect is `1/extinction` of the maximum, with the extinction
constant fixed at 3. So an object at the nominal blast radius takes a third of the damage,
not none — the radius is a characteristic distance, not a hard edge. A rebuild that uses a
cutoff instead will make every grenade in the shipped data feel weaker.

Attenuation through materials is **multiplicative per surface**, using the same
shoot-through factor the bullet system uses. That is how a blast is stopped by a wall, and
weakened but not stopped by a wooden fence — one table serves both weapons and explosions.

## `Explode` — the instant

**Contract** — begin the explosion. Requires a known initiator and a prepared position.
Everything after this runs over several frames.

```text
FUNCTION explode()
  REQUIRE an initiator is known, and the position has been set
  mark exploding; request unconditional per-frame updates
  OnBeforeExplosion()                       # hides the object, unless it is a smoke grenade
  play the explosion sound at the position
  place the decals
  create the particle effect, oriented with its up axis along the explosion's direction,
    carrying the exploding object's own velocity
  StartLight()

  # fragments — every one is a real bullet
  REPEAT frag_count times
    pick a uniformly random direction
    inject a bullet at the explosion position with that direction, the configured speed,
      damage, impulse, frag hit type and frag radius, attributed to the initiator,
      with hits sent only if this host is authoritative

  IF this object is remotely simulated THEN RETURN     # the blast wave is the owner's job

  # blast wave — collect once, apply over the following frames
  query the spatial index for every collidable within the blast radius
  keep every physics-holder among them except this object itself
  activate an expanding shape at the explosion position    # pushes bodies apart
  IF the player is within 30 metres THEN
    add a screen effector scaled by (30 - distance) / 30
```

**Invariants** — **fragments are bullets, not a computation.** Each one is injected into the
same bullet manager a weapon fires into, travels at a finite speed, is traced against the
same geometry and produces the same wallmarks and material sounds. That is why fragments
respect cover perfectly while the blast wave only respects it statistically, and it is why a
grenade's damage arrives over several milliseconds rather than at once.

The screen-shake radius is a compiled-in 30 units, independent of the blast radius. A small
firecracker and a rocket shake the screen over the same distance, scaled only by how close
the player is.

**Notes** — the "activate explosion box" step grows a physics shape at the explosion point
with the object's own collision temporarily disabled, which is how bodies at the centre get
pushed out rather than being trapped inside each other. Its cost is profiled explicitly,
which says it was once a problem.

## `UpdateCL` — the duration

**Contract** — advance the explosion by one frame. Runs only while exploding.

```text
FUNCTION update()
  IF not exploding THEN RETURN
  IF already marked exploded THEN
    stop asking for per-frame updates; clear the exploding flag; OnAfterExplosion(); RETURN

  IF the duration has run out AND every blasted object has been processed THEN
    mark exploded; StopLight()
  ELSE
    duration_left = duration_left - frame delta
    IF the object was NOT hidden at the start AND its hide delay has now elapsed THEN hide it
    UpdateExplosionPos(); UpdateExplosionParticles()
    ExplodeWaveProcess()                     # at most three objects this frame
    IF the light is on and within its lifetime THEN
      fade its colour and range linearly to zero over light_time
    ELSE StopLight()
```

**Invariants** — the explosion ends when **both** the timer has expired and the blasted-object
list is empty. A crowded explosion outlives its configured duration rather than dropping
damage on the floor.

There is a one-frame gap between "exploded" and the teardown: the flag is set in one frame
and acted on in the next. That gap exists so the last frame's light and particles are still
drawn.

### `ExplodeWaveProcess` and `ExplodeWaveProcessObject`

**Contract** — apply the blast to at most three objects per frame, taken from the back of the
list. Objects destroyed since the list was built are dropped first.

```text
FUNCTION process_one(target)
  IF the target has no visual THEN RETURN      # nothing to hit
  effect = ExplosionEffect(...)
  hit = blast_hit * effect; impulse = blast_impulse * effect
  IF either is above a small threshold THEN
    direction = normalize(target centre - explosion position), biased upward:
      add up_throw_factor to its vertical component, then renormalize
    send a hit event naming the initiator as the attacker and this object as the weapon,
      with the blast hit type and bone zero
```

**Invariants** — the budget of three per frame is the reason the whole thing has a duration
at all. Each object costs five ray queries through the collision database, so twenty objects
would be a hundred rays in one frame.

The upward bias is applied to a *normalized* direction and the result is renormalized with a
closed-form magnitude rather than a square root of the vector — the comment in the source
derives it. It matters because it keeps the impulse's magnitude exactly at one regardless of
the bias, so the impulse parameter means the same thing for a target above the explosion and
one beside it.

Damage is attributed with the **initiator** as the attacker and the **exploding object** as
the weapon. That pair is what makes a grenade kill count for the person who threw it.

## The event path

### `GenExplodeEvent`, `OnEvent`, `ExplodeParams`

**Contract** — an explosion is *requested* by an event and *performed* on receipt, so that
every host agrees on where and when it happened.

```text
FUNCTION request(position, normal)
  IF this host is not authoritative for this object THEN RETURN
  REQUIRE the event has not already been sent, and an initiator is known
  send an explode event carrying the initiator, the position and the normal
  mark event sent

FUNCTION on_event(explode)
  IF already exploding THEN ignore
  read the initiator, position and normal
  ExplodeParams(position, normal); Explode()
  duration_left = configured duration
```

**Invariants** — `ExplodeParams` raises the explosion position by a tenth of a unit, marked
in the source as a fake. It stops a grenade resting on the floor from detonating exactly at
the ground plane, where the decal and the light would be half buried and the upward rays
would immediately hit the floor.

### `FindNormal`

**Contract** — find the surface the object is lying on, for the explosion's orientation. Casts
one ray straight down over the object's own radius; uses the struck triangle's normal if it
hit static geometry, and straight up otherwise.

**Notes** — a hit on a *dynamic* object is treated the same as a miss. The comment says the
static case "should find the triangle and compute its normal", which is what the code does;
the dynamic case was never written.

## Lifecycle and cleanup

### `net_Destroy`, `net_Relcase`, `OnAfterExplosion`, `HideExplosive`, `Useful`

**Contract** — `OnAfterExplosion` stops any retained particle effect and destroys the object,
but only on the host that owns it. `HideExplosive` makes the object invisible, disabled, and
removes its physics assembly from the world — which is what "the grenade disappears when it
goes off" actually means.

`net_Relcase` handles the two ways an explosion can hold a reference to a dying object: the
initiator (cleared to "unknown", in single player only) and the blasted-object list (the
entry is erased).

**Notes** — `Useful` reports whether the explosion state is entirely untouched. It is the
reusability test for a pooled item: a grenade that has begun exploding can never be handed
out again.

The initiator resolution falls back to the exploding object itself when nobody is recorded,
which makes an unattributed explosion count as a suicide rather than crashing the kill
attribution.

## `StartLight`, `StopLight`, `LightCreate`, `LightDestroy`

**Contract** — a shadow-casting light created at the explosion position, fading its colour
and range linearly to zero over the configured light time, then released.

**Notes** — the light is created fresh at each explosion rather than being held. An assertion
that no light already exists is commented out, which suggests the retained form once leaked.

## `UpdateExplosionParticles`

**Contract** — for an explosion whose effect is marked dynamic, move the effect to the
object's current position each frame and hand it the implied velocity.

**Notes** — most explosions leave their effect where it started. A dynamic one is for
something that keeps moving while it burns — a fuel trail, a burning vehicle. The velocity
is derived by differencing positions rather than being read from the physics, which is
correct for an object being moved by anything, not only by the solver.

## `random_point_in_object_box`

**Contract** — a free function: a uniformly random point inside an object's bounding box, in
world space. Used by subclasses to answer "where inside me did the explosion start".

**Notes** — it transforms the random offset by the object's transform and *then* adds the
box's centre, which composes the two in the wrong order for an object whose box is not
centred on its origin. The shipped models are close enough to centred that it is not
visible.
