# src/xrGame/GraviZone.cpp

> The gravitational anomaly: an inner region that pulls everything toward its centre and an outer blowout that hits whatever reaches it, plus a telekinesis cycle that lifts inert objects into the air and drops them again.

**Needs** — [`GraviZone.h`](GraviZone.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`ai/monsters/telekinesis.h`](ai/monsters/telekinesis.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`Level.h`](Level.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`GraviZone.h`](GraviZone.h.md); callers name that, not this file.
**Tier floor** — T2: forces and impulses over a set of tracked objects; no device or layout concern

## Purpose

A zone (see the glossary: an anomaly is a first-class entity, not decoration) whose
effect is spatially split in two. Outside an inner fraction of the radius the zone
*pulls*; inside it, once the blowout timer fires, the zone *hits*. The split is what
makes the anomaly readable to a player — you are dragged in, and then something happens.

The pull is applied by two different mechanisms depending on the victim, and this
distinction is the load-bearing decision in the file: a living creature is moved by
injecting a velocity into its character movement controller, because a creature is not a
rigid body and an impulse on it would be ignored; a corpse or an inert object is moved by
a real impulse on its rigid body.

The file is written as an abstract base plus a trivial concrete class. The base does all
the work but leaves the telekinesis controller abstract, so that another anomaly kind can
reuse the same field behaviour while sharing one telekinesis controller with something
else. The concrete zone simply owns its own.

## State

```text
RECORD GraviZoneConfig                      # all read from the entity's section
  throw_in_impulse        : real   # pull strength for inert bodies, calibrated for 100 kg
  throw_in_impulse_alive  : real   # pull strength for living creatures
  throw_in_atten          : real   # loaded, never read — see Notes
  blowout_radius_percent  : real   # fraction of radius separating "pull" from "blowout"
  tele_height             : real   # how high telekinesis lifts an object
  time_to_tele            : int    # how long a lift lasts, in milliseconds
  tele_pause              : int    # gap between one lift ending and the next beginning
  tele_particles_big      : optional<text>   # effect for objects above the small-object radius
  tele_particles_small    : optional<text>   # effect for objects below it

RECORD GraviZoneState
  tele_time  : int    # milliseconds accumulated in the current lift/pause cycle
```

Invariant: the telekinesis cycle runs only while the zone is idle. Entering blowout
releases every lifted object at once, so an object is never both held aloft and being
hit.

## `Load`

**Contract** — reads every field above after the base zone has read its own. The two
particle names are optional; absent, the corresponding size class simply plays no lift
effect. No defaults are supplied in code — the shipped sections must carry the
non-optional keys or loading fails.

## `Affect`

**Contract** — the per-object effect hook, called by the base zone once per tracked object
per effect tick. Decides pull versus hit from the object's distance and from the blowout
timer, and applies exactly one of them.

**Invariants** — an object outside the inner fraction is always pulled, never hit. An
object inside it is pulled *to* the boundary rather than to the centre until the blowout
moment arrives, which is why a victim visibly hangs at a fixed radius before the
explosion rather than being crushed into a point.

```text
FUNCTION affect(object_info)
  target = object_info.object as a physics-shell holder
  IF target is none THEN RETURN                  # zones only act on things with a body

  direction = normalize(throw_in_centre() - target.centre)
  IF the two centres coincide THEN direction = straight up   # avoid an undefined direction
  distance = |throw_in_centre() - target.centre|
  fraction = distance / radius

  IF fraction > blowout_radius_percent AND target is locally simulated THEN
    pull(target, direction, distance)
    RETURN

  # inside the inner region, or an object owned by another host
  IF the blowout moment does not fall inside this tick's time window THEN
    # not yet: hold it at the boundary instead of dragging it further in
    pull(target, direction, blowout_radius_percent * radius)
    RETURN

  throw(object_info, target, direction, distance)
```

**Notes** — "locally simulated" gates the pull because in multiplayer only the host that
owns an object may move its body; a remote object still receives the hit, which is
replicated, but not the pull, which is not.

## `AffectPull` / `AffectPullAlife` / `AffectPullDead`

**Contract** — `AffectPull` dispatches on whether the victim is a living creature. The
living path injects a control velocity into the creature's movement controller; the dead
or inert path applies an impulse to the rigid body. Both are one-shot per tick and
neither blocks.

```text
FUNCTION pull(target, direction, distance)
  IF target is a living creature THEN
    strength = relative_power(distance, radius)      # 1 at the centre, 0 at the rim
    velocity = direction * throw_in_impulse_alive * strength^5
    target.movement.add_control_velocity(velocity)
  ELSE IF target has a rigid body THEN
    target.body.apply_impulse(direction,
                              distance * throw_in_impulse * target.mass / 100)
```

**Notes**

- The fifth power on the creature falloff is the file's most opinionated number. It makes
  the pull almost nothing across most of the radius and overwhelming near the centre, so
  a player can walk past the rim of the anomaly and is helpless once committed. A gentler
  exponent makes the anomaly feel sticky everywhere, which is the wrong feel.
- The inert falloff runs the *other* way: it is proportional to distance, so a distant
  object is yanked hard and a near one gently, which settles debris toward the centre
  instead of oscillating it through.
- Dividing by 100 makes the configured impulse read as "for a 100-kilogram object", which
  is how the tuning values in the shipped data were authored.

## `AffectThrow`

**Contract** — issues a hit against the victim: a damage value derived from the distance
falloff, an impulse scaled by that damage and the victim's mass, and the blowout hit
type. Hits below a small threshold are skipped entirely, along with their particle
effect, so the rim of the zone produces no cosmetic noise. The hit is routed through the
engine's hit-creation path rather than applied directly, so it replicates and so that
armour, hit-type resistance and death handling all run.

## `IdleState`

**Contract** — extends the base zone's idle tick with the telekinesis cycle. Runs only
while the zone is genuinely idle; the moment the base reports otherwise every lifted
object is released.

```text
FUNCTION idle_state() -> bool
  still_idle = base.idle_state()
  tele_time = tele_time + frame_milliseconds

  IF NOT still_idle THEN
    telekinesis.release_all()
    RETURN still_idle

  IF tele_time > time_to_tele THEN
    FOR EACH tracked object THAT is held by telekinesis
      telekinesis.release(object)
      stop_lift_particles(object)

  IF tele_time > time_to_tele + tele_pause THEN
    tele_time = 0
    FOR EACH tracked object THAT has a rigid body AND is not held
      telekinesis.hold(object, strength: 0.1, height: tele_height, duration: time_to_tele)
      start_lift_particles(object)

  RETURN still_idle
```

**Notes** — the two thresholds are deliberately checked in the same tick so that on the
frame the pause ends, everything is released and then immediately re-grabbed; the visual
result is a slow collective rise and fall of every loose object in the zone, which is the
anomaly's signature.

## `BlowoutState`

**Contract** — extends the base blowout tick by running the blowout's own update and
applying the per-object effect pass. Returns whatever the base decided about the state.

## `CheckAffectField`

**Contract** — answers "is this object in the pulling region rather than the blowout
region", given its distance expressed as a fraction of the radius. Overridable so a
derived anomaly can make the boundary depend on the victim — a large creature can be made
to reach the blowout sooner than a small one.

## `ThrowInCenter`

**Contract** — yields the point everything is pulled toward. The base answer is the
zone's own centre; it is a separate overridable step because a derived anomaly may pull
toward a moving or offset focus.

## `PlayTeleParticles` / `StopTeleParticles`

**Contract** — start or stop the lift effect on one object, choosing between the "big"
and "small" effect names by comparing the object's radius against the engine-wide
small-object threshold. Silently does nothing when the object cannot host particles or
when the chosen effect name was not configured. The effect is attached at a fixed offset
one unit above the object's origin and tagged with this zone's identifier, so that the
same object lifted by two zones does not have one zone's stop cancel the other's effect.

## `net_Spawn` / `net_Destroy` / `net_Relcase` / `shedule_Update`

**Contract** — the lifecycle hooks, each doing one thing beyond delegating to the base:
destruction releases every telekinesis hold before the base tears the zone down; the
scheduled update ticks the telekinesis controller; the reference-release hook drops any
link the controller holds to an object that is being destroyed. That last one is the
invariant from the conformance list — a destroyed entity must be unreferenced by every
subsystem before its memory goes away — and telekinesis is exactly the kind of
long-lived cross-entity reference that violates it if forgotten.

**Notes** — the spawn hook is a pure delegation and exists only because the class
declares it; it carries no decision.

## `CGraviZone`

**Contract** — the concrete anomaly: the base's behaviour with a telekinesis controller
of its own. Nothing else.

**Notes** — the abstract-controller split exists so that a creature which telekinetically
throws objects can share the field behaviour while routing through *its* controller. The
split is otherwise arbitrary and a rebuild may collapse it.

## Could not recover

- The `throw_in_atten` parameter is read from every shipped section and never consulted.
  It presumably once shaped the pull falloff that is now a hard-coded fifth power.
