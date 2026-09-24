# src/xrGame/RadioactiveZone.cpp

> The radiation anomaly: a zone that delivers a steady, distance-scaled dose in fixed time quanta rather than in discrete hits.

**Needs** — [`RadioactiveZone.h`](RadioactiveZone.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`Hit.h`](Hit.h.md) · [`game_base_space.h`](../xrServerEntities/game_base_space.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: accumulation arithmetic and a distance falloff

## Purpose

Every other anomaly in the game punishes an intrusion: you step in, something happens, you
are thrown or burned or crushed. Radiation is continuous, and that single difference is
what this class exists for. It overrides the generic zone's damage application to
*integrate* dose over time in fixed quanta, so that a player who walks through quickly
takes proportionally less than one who lingers, independent of frame rate.

It also carries the multiplayer variant of the same idea, which must run on the server and
reach clients as explicit hit events rather than as locally computed damage.

## State

`Stateless.` Everything it needs is the generic zone's: the set of objects currently felt
inside the shape, each with the clock time up to which it has already been dosed.

**Invariants** — that per-object "already dosed up to" time is what this whole file turns
on. It advances in whole quanta and never runs ahead of the current time, and it is
clamped from below so it can never lag more than a bounded number of quanta behind.

## `Affect`

**Contract** — called by the generic zone for one object inside it, as often as the zone
updates. Computes the object's dose rate from its distance to the zone centre against the
zone's falloff, then emits one damage event per elapsed quantum, advancing the object's
dosed-up-to time by one quantum each time. Emits nothing and returns immediately when less
than one quantum has passed, or when the dose rate at this distance is negligible.

```text
CONSTANT quantum = 0.1 seconds

FUNCTION Affect(info)
  IF info.object IS none THEN RETURN
  now <- global_clock
  IF info.dosed_until + quantum > now THEN RETURN      # not a full quantum yet

  info.dosed_until <- clamp(info.dosed_until, now - 3*quantum, now)   # bound the catch-up

  centre <- zone_transform applied to collision_shape.sphere_centre
  rate   <- falloff(distance(info.object.position, centre), nearest_shape_radius(info))
  IF rate < epsilon THEN
    info.dosed_until <- now                            # mark as current so it is not re-tested
    RETURN

  dose_per_quantum <- rate * quantum
  WHILE info.dosed_until + quantum < now
    deliver_hit(target: info.object, source: self,
                direction: zero, power: dose_per_quantum,
                bone: none, impulse: 0, kind: radiation)
    info.dosed_until <- info.dosed_until + quantum
```

**Invariants**

- The clamp is the interesting line and it is not a safety check. A zone whose update was
  starved — the player was elsewhere, the level was loading, the frame hitched — would
  otherwise emit one hit per quantum for the whole gap, which for a ten-second gap is a
  hundred hits in one frame and a dead player. Capping the backlog at three quanta makes
  the worst case three hits and makes long absences cost nothing, which is the correct
  gameplay answer: you were not standing there.
- The dose is delivered as many small hits rather than one accumulated hit because the
  damage pipeline is per-hit: armour resistance, script callbacks, and the player's
  radiation meter all key off individual hit events, and a single large hit would resolve
  differently through the armour curve.
- The direction and impulse are zero. Radiation must not push, and a zero direction is what
  tells the hit pipeline there is no directional component to resolve against armour zones.

**Notes** — the hit type is the zone's own configured type rather than a hard-coded
radiation type. A source comment records that it used to be hard-coded; making it
configurable is what lets the same class serve any continuous-dose anomaly.

## `UpdateWorkload`

**Contract** — the multiplayer path. On a server running any mode other than single
player, and only while the zone is enabled, every actor currently inside gets a dose for
the *elapsed interval* — not in quanta — computed from its distance and falloff and scaled
by the interval. The dose is delivered by constructing a hit event message and handing it
to the object's event handler directly, rather than by calling the damage path.

```text
FUNCTION UpdateWorkload(elapsed_ms)
  IF enabled AND game_mode != single_player THEN
    centre <- zone_transform applied to collision_shape.sphere_centre
    FOR EACH info IN objects_inside
      IF info.object IS being destroyed THEN CONTINUE
      IF info.object IS NOT an actor THEN CONTINUE
      power <- falloff(distance(info.object.position, centre),
                       nearest_shape_radius(info)) * elapsed_ms / 1000
      event <- hit_event(target: info.object.id, source: self.id, weapon: self.id,
                         direction: zero, power: power, bone: none,
                         impulse: 0, kind: self.hit_type)
      info.object.handle_event(event)
  base.UpdateWorkload(elapsed_ms)
```

**Invariants** — only actors are dosed in multiplayer. Creatures and items inside a
radiation zone take nothing, because the multiplayer modes have no creatures and the
bookkeeping would be replicated for nothing.

**Notes** — the round trip through a serialized event message delivered to the object in
the same process is not an accident and should survive a rebuild in spirit: on a server,
*every* authoritative state change must take the same route it would take to a remote
client, so that the local player and a remote one see identical damage. Taking a shortcut
here is how a server and its clients diverge. A rebuild should keep the "authoritative
changes are events" rule and need not keep the literal serialize-then-immediately-parse.

The interval scaling here divides by a thousand because the scheduler's interval arrives
in milliseconds while the falloff yields a per-second rate.

## `feel_touch_new`

**Contract** — when an object first enters the zone, the generic zone's handling runs. In
multiplayer, an entering *actor* additionally receives a zero-power hit of the zone's
type. A hit with no damage exists only for its side effects: it is what makes the client's
radiation indicator light up the moment the player crosses the boundary, rather than up to
one update interval later.

## `feel_touch_contact`

**Contract** — decides whether a candidate object is considered inside. Only actors are;
everything else is rejected outright. An actor must both actually intersect the zone's
shape and itself accept contact from this zone — the actor has the final say, which is how
protective equipment and script overrides exclude a player from a zone without moving him.

**Notes** — rejecting non-actors here is why `Affect` sees only actors in practice even in
single player, despite its general phrasing. A rebuild wanting radiation to affect
creatures changes this one predicate.

## `BlowoutState`

**Contract** — asks the generic zone whether it is currently in its blowout (discharge)
phase, and if it is *not*, still advances the blowout animation one step before reporting
false. The effect is that a radiation zone's visual discharge keeps ticking during its
idle phase instead of freezing between discharges — a radiation field has no dramatic
"fires and recharges" rhythm, so its particles must run continuously.

## `nearest_shape_radius`

**Contract** — the falloff radius to use for an object. A zone built from a single shape
reports the zone's own radius. A zone built from several shapes reports the radius of the
*first* shape.

**Notes** — the name promises the nearest shape and the code takes the first one, and
nothing chooses between them. For the shipped single-shape zones the two agree, which is
presumably why it was never finished. **Could not recover**: the intended behaviour for a
multi-shape radiation zone. A rebuild should implement what the name says — pick the shape
whose surface is closest to the object — or collapse multi-shape zones to one radius and
delete the branch.

## `Load`

**Contract** — reads the zone's tuned parameters from its configuration section. Adds
nothing to the generic zone's load; the override exists only as a hook.
