# src/xrGame/agent_location_manager.cpp

> The squad's shared opinion of places: which spots are dangerous and for how long, and which cover point a member may claim without crowding a comrade.

**Needs** — [`agent_location_manager.h`](agent_location_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_enemy_manager.h`](agent_enemy_manager.h.md) · [`danger_location.h`](danger_location.h.md) · [`cover_point.h`](cover_point.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — reached through its declarations in [`agent_location_manager.h`](agent_location_manager.h.md); callers name that, not this file.
**Tier floor** — T2: per-cover-point scoring inside the AI's frame budget

## Purpose

Two squad-level facts about *places*, which no individual member can hold on its own.

The first is **danger**: a grenade landed here, a comrade died there. A danger is a
position, a radius, a start time, a duration, and a mask saying which squad members it
applies to. It decays: a spot is most dangerous the instant it becomes dangerous and least
dangerous just before it expires.

The second is **cover allocation**: squad members must not all take cover behind the same
crate. Without a shared arbiter each member independently scores cover points and they
converge on the same best one; this manager is that arbiter.

## State

```text
RECORD DangerLocation          # defined elsewhere; the fields this manager reasons about
  position    : vector
  radius      : real
  start_time  : int            # global clock
  interval    : int            # milliseconds it stays dangerous
  mask        : squad mask     # which members this danger applies to

RECORD LocationManager
  dangers : list<DangerLocation>   # shared, reference-counted; members hold them too
  squad   : agent
```

Invariants:

- Dangers are identified **by position**, not by the object that caused them. Two grenades
  landing in the same spot are one danger, merged. That is why adding is a merge rather
  than an append.
- A danger is reference-counted and handed out to members, who hold it while reacting; the
  manager's removal drops its own reference, not everyone's.
- The mask is how one squad's danger applies to some of its members and not others — a
  member who cannot see the grenade should not flinch from it.

## `add`

**Contract** — registers a danger. First notifies every squad member the danger's mask
selects, so each can react immediately. Then merges: if no danger exists at that position
the new one is kept; otherwise the existing one's start time is *replaced* by the new
one's and its duration and radius are each widened to the larger of the two.

```text
FUNCTION add(danger)
  FOR EACH member IN squad.members WHERE danger.mask selects member
    member.on_danger_location_added(danger)

  existing = danger_at(danger.position)
  IF existing is none THEN
    dangers.append(danger)
    RETURN

  existing.start_time = danger.start_time         # the clock restarts
  existing.interval   = max(existing.interval, danger.interval)
  existing.radius     = max(existing.radius,   danger.radius)
```

**Invariants** — the merge is *monotone in extent but resetting in time*: a second grenade
in the same spot restarts the danger and can only widen it, never narrow it. That is the
behaviour a squad needs — the spot is dangerous again from now — and it means a stream of
explosions in one place keeps the area avoided without the list growing.

**Notes** — notification happens before the merge, so members are told about the *new*
danger even when it is folded into an existing one. That is correct: the reaction is to
the event, not to the record.

## `danger`

**Contract** — scores a cover point for one member: how *safe* it is, as a multiplier in
the unit interval where one is completely safe. Every live danger that applies to this
member and whose radius reaches the point multiplies the result by the fraction of its
duration already elapsed.

```text
FUNCTION safety(cover_point, member) -> real
  result = 1
  mask = squad.mask_of(member)
  FOR EACH danger IN dangers
    IF now > danger.start_time + danger.interval THEN CONTINUE     # expired
    IF NOT danger.mask.applies_to(mask) THEN CONTINUE
    IF 1 + distance(danger.position, cover_point) > danger.radius THEN CONTINUE
    result = result * (now - danger.start_time) / danger.interval
  RETURN result
```

**Invariants**

- The elapsed fraction runs from zero at the moment the danger appears to one as it
  expires, so a fresh danger scores a nearby cover point at zero — completely unusable —
  and an old one barely penalizes it. This is the decay, and it is *linear in time* and
  **independent of distance**: a point just inside the radius is penalized exactly as much
  as the point at the centre. A rebuild is free to fall off with distance; the shipped
  tuning was done against the cliff.
- Multiple overlapping dangers *multiply*, so two half-decayed dangers leave a quarter of
  the safety. That makes a spot covered by several dangers far worse than the worst of
  them, which is the intent.
- The distance test adds one metre before comparing to the radius, which shrinks the
  effective radius by a metre. There is no stated reason; it reads as a guard against a
  point sitting exactly on the boundary.

## `suitable`

**Contract** — may this member claim this cover point? Rejects it if it would crowd
another squad member, and optionally if a known enemy is standing on top of it. The rules,
per other member:

```text
FUNCTION suitable(member, point, consider_enemies) -> bool
  FOR EACH other IN squad.members, excluding member
    IF other has no cover claimed THEN
      IF other is registered in combat THEN CONTINUE      # it will claim one shortly; don't block on it
      IF distance(other.position, point) <= 5 THEN RETURN false   # standing there already
      CONTINUE
    IF distance(other.cover, point) <= 5 THEN
      # too close to a claimed cover: yield only if the incumbent is the better fit
      IF distance(other.position, other.cover) <= distance(member.position, point) + 2 THEN
        RETURN false

  IF consider_enemies THEN
    FOR EACH enemy known to the squad
      IF distance(enemy.position, point) < 3 THEN RETURN false

  RETURN true
```

**Invariants**

- Five metres is the crowding radius for cover — two members closer than that are in each
  other's way. It appears three times and is one concept.
- The tie-break when two covers are close is *who has less far to travel*, with a
  two-metre bias in favour of the incumbent. The bias is hysteresis: without it two members
  at nearly equal distance would repeatedly displace each other.
- A member who has not yet claimed cover but is *in combat* is ignored rather than treated
  as an obstacle. It is about to claim something, and treating its current standing
  position as reserved would deadlock the allocation.
- The enemy exclusion radius is three metres, reduced at some point from ten (the original
  value survives in the source). A rebuild should treat it as tuning.

## `make_suitable`

**Contract** — commits the claim: records the cover point against the member, then evicts
every *other* member whose claimed cover is within the crowding radius, telling each that
its cover was blocked and clearing its claim. Passing no point releases the member's claim
without evicting anyone.

**Invariants** — eviction is unconditional here, with none of the distance tie-break that
`suitable` applies. The asymmetry is deliberate: `suitable` decides whether a claim is
*allowed*, and once allowed the claim wins outright. An evicted member is notified so it
can look for another point rather than standing where it thinks it is covered.

## `remove_old_danger_covers`

**Contract** — the per-update sweep. Drops every danger that has outlived its usefulness,
notifying each squad member the danger's mask selected that it is gone, so members holding
a reaction to it can stop.

**Invariants** — the notification happens inside the removal test, which is a predicate
with a side effect. It works because the sweep visits each element once, but it is fragile
against any change to how the list is compacted. A rebuild should partition first and
notify second.

## `update`

**Contract** — runs the expiry sweep. Unlike its sibling managers this one does have
per-frame work, because danger decays on a clock.

## `location` (by position, by object) / `locations` / `clear` / `remove_links`

**Contract** — look a danger up by its position or by the object that caused it; read the
whole list; empty it; and drop everything naming an object that is leaving the simulation.
The by-position lookup is what makes `add` a merge.

**Notes** — the two lookups compare against different things through the same equality
operation on the danger record, which is overloaded for both a position and an object.
That overloading is why a danger can be found either way; a rebuild should give the two
queries distinct names.
