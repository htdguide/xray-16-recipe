# src/xrGame/ai/monsters/anomaly_detector.cpp

> Makes a creature that has touched an anomaly path around it for the next half minute, by turning the anomaly into a temporary movement restrictor.

**Needs** — [`anomaly_detector.h`](anomaly_detector.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`restricted_object.h`](../../restricted_object.h.md) · [`CustomZone.h`](../../CustomZone.h.md) · [`space_restriction_manager.h`](../../space_restriction_manager.h.md) · [`Level.h`](../../Level.h.md)
**Used by** — [`anomaly_detector.h`](anomaly_detector.h.md)
**Tier floor** — T3: a short list, two set operations against the pathfinder's restrictor set, and a timed sweep

## Purpose

A creature has no way to see an anomaly. The only evidence available is contact: it walked
into one. This component turns that contact into navigation knowledge, by adding the
anomaly's volume to the creature's own **out-restrictor** set — the volumes the pathfinder
must route around — for a configured period, after which the creature forgets and may walk
in again.

The forgetting is deliberate. A creature that permanently learned every anomaly would end
up with a pathfinding cost model nobody authored, and a level's creature population would
drift into unnatural corridors over a long session.

The component is *off by default* and switched on by whichever creature behaviour wants it,
because running a proximity sweep every scheduled update is not free.

## State

```text
RECORD AnomalyDetector
  creature       : reference
  radius         : real = 15      # detection sweep radius, from configuration
  memory_period  : int  = 30000   # ms an anomaly stays avoided, from configuration
  enabled        : bool
  remembered     : list<{ zone : reference, registered_at : int }>
```

**Invariants** — `registered_at` of zero is a sentinel meaning "seen, not yet applied". An
entry is created with zero by the contact notification and stamped with the real time by
the next scheduled pass, which is when the restriction is actually added. So an entry's
lifetime is exactly `memory_period` from the *pass that applied it*, not from the contact.

An entry is never duplicated: a second contact with an already-remembered zone is dropped.

**Configuration** — `Anomaly_Detect_Radius` and `Anomaly_Detect_Time_Remember` on the
creature's section, both optional with the defaults above.

## `on_contact`

**Contract** — called when the creature's touch sense reports a nearby object. Records the
object if and only if it is a zone, the zone declares itself a restrictor, that restrictor
is not *already* in the creature's restriction set, and it is not already remembered.
Otherwise does nothing. Silently ignores everything while disabled.

```text
FUNCTION on_contact(object)
  IF NOT enabled THEN RETURN
  IF object is not a zone THEN RETURN
  IF zone.restrictor_type = none THEN RETURN
      # some zones are effects with no navigation volume; nothing to route around
  IF zone is already among the creature's current restrictions THEN RETURN
      # it was placed there by the level or by script; not ours to manage
  IF zone is already remembered THEN RETURN
  remembered.append({ zone, registered_at: 0 })
```

**Invariants** — the "already a restriction" test is what keeps this component from
fighting the level's own authored restrictions. It must only ever add and remove the
restrictions it itself introduced; anything already present belongs to somebody else and
removing it would silently widen where the creature may go.

## `update_schedule`

**Contract** — run once per scheduled update. When enabled, drives the creature's touch
sense over the detection radius, which is what produces the contact notifications. Then
applies pending restrictions, removes expired ones, and drops expired entries. Allocates two
scratch lists per call.

```text
FUNCTION update_schedule()
  IF enabled THEN creature.touch_sweep(creature.position, radius)

  IF remembered is empty THEN RETURN

  to_add = [ e.zone.id FOR e IN remembered WHERE e.registered_at = 0 ]
  stamp each of those entries with the current time
  pathfinder.restrictions.add(out: none, in: to_add)

  to_remove = [ e.zone.id FOR e IN remembered WHERE e.registered_at + memory_period < now ]
  pathfinder.restrictions.remove(out: none, in: to_remove)

  drop every entry whose registered_at + memory_period < now
```

**Invariants** — add and remove happen in the same pass and in that order, so a zone
registered and expired within one period is never left applied. The expiry test in the
removal and in the drop is the same test, which is what keeps the restriction set and the
list in step.

**Notes** — this is the component's one real bug surface and it is worth stating: the
restrictions are added to the **in**-restrictor channel, not the out-restrictor one. An
in-restrictor is a volume the creature must *stay inside*; an out-restrictor is one it must
stay out of. Adding an anomaly as an in-restriction intersects the creature's permitted
space with the anomaly's volume, which is the opposite of avoidance. The out list is
constructed and passed empty at both call sites. Whether the two channels are swapped in
this component or the naming is inverted relative to the restrictor layer is not
recoverable from this file alone; a rebuild must check the restriction manager's semantics
before copying this.

The touch sweep is driven from here rather than from the creature's own perception, which
means the sweep radius for *this purpose* is independent of the creature's other senses.
That is the reason the radius is a separate setting.

## `load` / `reinit`

**Contract** — `load` reads the two settings from the creature's section with defaults.
`reinit` clears the remembered list and switches the component off; it is called on spawn
and on level reload. Note that it does **not** remove outstanding restrictions from the
pathfinder — the pathfinder's own restriction set is reinitialised alongside it by the
creature, which is the only reason that is safe.

## `enable` / `disable`

**Contract** — the on/off switch. Disabling stops the sweep and stops new contacts being
recorded, but leaves the existing memory to expire normally, so a creature that stops
detecting still finishes routing around what it already knows.
