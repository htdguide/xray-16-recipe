# src/xrGame/Mincer.cpp

> The meat-grinder anomaly: a gravity zone that lifts everything caught inside it into a whirlwind, holds them there, and tears them apart when it discharges.

**Needs** — [`Mincer.h`](Mincer.h.md) · [`GraviZone.h`](GraviZone.h.md) · [`TeleWhirlwind.h`](TeleWhirlwind.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`PHDestroyableNotificate.h`](PHDestroyableNotificate.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`ParticlesPlayer.h`](ParticlesPlayer.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`Actor.h`](Actor.h.md) · [`Hit.h`](Hit.h.md) · [`Level.h`](Level.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`Mincer.h`](Mincer.h.md)
**Tier floor** — T3: a zone state machine driving a telekinesis helper; all the numeric work is delegated

## Purpose

An anomaly cycles through idle, accumulation, blowout and discharge. Most anomalies express
the blowout as a force applied to whatever is inside. This one instead *takes hold* of
everything inside using the telekinesis helper — the same machinery a psychic creature uses to
lift objects — suspends them at a fixed height, and releases them destructively at the moment
of discharge.

That is the whole design: the zone owns a telekinesis controller whose lifetime is tied to the
blowout phase, and whose release is what kills.

## State

```text
RECORD Mincer                        # on top of the base gravity zone
  telekinetics       : Telekinesis   # holds the suspended objects
  torn_particles     : text          # the effect played on a body as it comes apart
  tearing_sound      : sound
  actor_blowout_radius_percent : real   # default 0.5; see BlowoutRadiusPercent
```

**Invariants** — an object held by the telekinesis controller must be released before the zone
is destroyed, or the controller outlives the objects it references. The destroy path clears
the impact list for exactly this reason.

The controller's centre is the zone's position raised by the configured suspension height, and
it is set once at spawn — the mincer does not move.

## `Load`

**Contract** — reads the tearing particle effect, the throw-out impulse, the torn-body
particles, the body-tearing sound and the actor blowout radius fraction from the section, on
top of the base zone's load. Every key is required.

**Notes** — two separate particle effects are configured: one the telekinesis controller plays
while destroying an object, one this class plays on the body itself. They are different
because one belongs to the whirlwind and one to the victim.

## `net_Spawn`

**Contract** — spawns as a gravity zone, then positions the telekinesis controller at the zone
centre raised by the suspension height and tells it who owns it.

**Invariants** — the controller must know its owner before it can apply a hit, because the hit
is attributed to the zone, not to the physics.

## `net_Destroy`

**Contract** — releases every telekinetic hold, then destroys as a gravity zone. Order matters:
the holds reference objects that the base teardown may release.

## `OnStateSwitch`

**Contract** — the state machine's one rule. Entering the blowout phase takes hold of every
object currently inside the zone; leaving it releases everything and clears the controller.

```text
FUNCTION OnStateSwitch(new_state)
  IF was not blowing out AND new_state is blowout THEN
    FOR EACH object recorded as inside the zone
      IF it has a physics body THEN
        telekinetics.activate(object, throw_in_impulse, suspension_height, HUGE_TIME)
      END IF
    END FOR
  END IF
  IF was blowing out AND new_state is not blowout THEN
    telekinetics.clear_deactivate()      # let everything go, without destroying it
  END IF
  inherited OnStateSwitch(new_state)
```

**Invariants** — the hold is registered with an effectively unbounded duration. The telekinesis
controller normally times its holds out; here the zone's own state machine is the timer, so the
controller must never release on its own. A rebuild must express "held until told otherwise"
rather than pick a large number.

**Notes** — the transition checks are written against the *old* state because the base call
that updates it comes last. That ordering is load-bearing: doing the base call first would make
both conditions false.

## `feel_touch_new`

**Contract** — an object entering the zone. If the zone is already blowing out and has not yet
reached its discharge instant, the newcomer is grabbed immediately rather than waiting for the
next cycle.

**Invariants** — the grab is refused once the discharge instant has passed, because everything
held is about to be destroyed and a late arrival would be killed without ever being lifted.

## `feel_touch_contact`

**Contract** — narrows the base zone's interest to objects that have a physics body. An object
the telekinesis controller cannot lift has no business being tracked.

## `BlowoutState`

**Contract** — the per-tick blowout behaviour. Detects the single tick in which the discharge
instant is crossed and, on that tick only, releases the telekinesis controller *destructively*.

```text
FUNCTION BlowoutState() -> bool
  result = inherited BlowoutState()
  # Fire exactly once, on the tick whose interval contains the discharge instant.
  IF discharge_instant is within (previous_state_time, current_state_time] THEN
    telekinetics.deactivate()        # the destructive release
  END IF
  RETURN result
```

**Invariants** — the edge test against the previous and current state times is what makes this
fire once. A test against the current time alone would fire on every tick after the instant,
killing repeatedly.

**Notes** — the two releases are different operations and the distinction is the heart of the
class: `clear_deactivate` lets go, `deactivate` destroys. The first is what happens if the
zone's cycle is interrupted; the second is the kill.

## `NotificateDestroy`

**Contract** — called back by the destructible-object machinery when something the controller
was holding comes apart. Plays the torn-body particles on the victim and the tearing sound at
the whirlwind centre, then issues an explosion-type hit to the victim carrying the impulse the
controller had accumulated for it.

```text
FUNCTION NotificateDestroy(notification)
  victim = notification.physics holder
  dir, impulse = telekinetics.draw_out_impact()     # consumes the recorded impact
  IF victim can play particles AND torn_particles is configured THEN
    victim.start_particles(torn_particles, upward, attributed to this zone)
  END IF
  play tearing_sound at telekinetics.centre
  issue hit(target: victim, source: this zone, direction: (1,0,1), power: 0,
            bone: none, position: bone origin, impulse: impulse,
            type: explosion)
```

**Notes** — the hit's *power is zero*. The kill has already happened; this hit exists only to
deliver the impulse that flings the pieces, and an explosion-type hit is how an impulse with no
damage is expressed. A rebuild that treats power zero as "no hit" will produce bodies that come
apart and then sit still.

The throw direction is a fixed diagonal rather than anything derived from the victim's
position. That is visible in play — pieces always fly the same way — and looks like an
unfinished detail rather than a decision.

## `AffectPullAlife`

**Contract** — the per-tick pull applied to a living entity inside the zone. Non-player
creatures take a damaging hit scaled by their distance from the centre; the player takes none
here, only the pull.

```text
FUNCTION AffectPullAlife(entity, direction, distance)
  power = falloff(distance, radius)
  IF entity is NOT the player THEN
    issue hit(target: entity, source: this zone, direction, power,
              bone: none, impulse: 0, type: the zone's configured blowout hit type)
  END IF
  inherited AffectPullAlife(entity, direction, distance)
```

**Invariants** — the player is exempt from the pulling damage, not from the zone. The player is
killed by the discharge, through the telekinesis path, not by attrition on the way in. That
asymmetry is what makes the anomaly survivable if the player escapes before discharge and
lethal if they do not.

## `BlowoutRadiusPercent`

**Contract** — how far from the centre the blowout reaches, as a fraction of the zone radius,
with a separate value for the player.

**Notes** — the player gets their own fraction (defaulting to half) so the anomaly's lethal
radius can be tuned for the player without changing how it treats creatures. This is the same
kind of difficulty accommodation as the hit-probability roll in the bullet manager, and a
rebuild that unifies the two radii will make the anomaly either unfairly large or visibly
harmless to creatures.

## `ThrowInCenter` · `Center`

**Contract** — where objects are drawn toward (the whirlwind centre, at the zone's own ground
height) and where the zone is (its position). The two differ in height only: objects are pulled
horizontally toward the whirlwind axis and lifted by the telekinesis controller, not pulled
diagonally toward a raised point.

## `AffectThrow` · `AffectPullDead`

**Contract** — a pure delegation to the base zone, and a deliberate no-op. A corpse is not
pulled: it is already inert, and pulling it would fight the ragdoll.
