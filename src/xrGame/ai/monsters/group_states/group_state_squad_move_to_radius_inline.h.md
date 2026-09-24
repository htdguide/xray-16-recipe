# src/xrGame/ai/monsters/group_states/group_state_squad_move_to_radius_inline.h

> Two ways to close to a ring around the enemy: fan the pack out by squad index across an arc on
> the home side, or simply approach straight in.

**Needs** — [`group_state_squad_move_to_radius.h`](group_state_squad_move_to_radius.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../state.h`](../state.h.md) · [Level graph — chapter 14](../../../../xrAICore/README.md)
**Used by** — [`group_state_squad_move_to_radius.h`](group_state_squad_move_to_radius.h.md)
**Tier floor** — T3: polar arithmetic per member, plus one navigation-mesh validity probe

## Purpose

These two states are what the player actually sees when a pack circles: creatures arriving at a
consistent distance from them, spaced out rather than stacked, and all on the same side. The pack
attack brain uses the fanned variant for its first stalking rung and the radial variant for the
two inner rungs (see [`group_state_attack_inline.h`](group_state_attack_inline.h.md)).

The two differ in exactly one computation — where on the ring to aim — and the difference is the
whole reason both exist.

## State

Both own an instance of the shared *move to a point, extended* parameter record, and hand it to
the base state machinery as the record the composite above will fill in. The fields that matter
here:

```text
  completion_dist : real    # doubles as the ring's radius; written by the composite
  point           : vector  # the computed point on the ring; rewritten every tick
  vertex          : int     # a navigation hint; written by the composite, usually absent
  action          : the gait, sounds and animation flags
  time_to_rebuild : int
  accelerated, braking, accel_type
```

**`completion_dist` carries two meanings at once** — it is both the radius of the ring the state
aims at and the distance at which arrival is judged. That conflation is what lets one parameter
record serve both this state and the ordinary move-to-point states, and it is load-bearing: a
rebuild that separates them must write the same number into both.

## `CStateGroupSquadMoveToRadiusEx` — the fanned variant

**Contract** — prime the path builder on entry. Each tick compute this member's point on the ring,
request the parameterised gait and animation flags, hand the point to the path builder with
cover-preferring paths and a 2-unit approach margin, apply the acceleration profile if asked, and
play the parameterised sound.

```text
FUNCTION compute_point()
  IF NOT in an active squad OR I have no index in it
    point = enemy.position                       # degrade to walking at the enemy
    RETURN

  members = squad.living_count()
  # divide a 90-degree arc among the members, by index
  arc_share = (PI - PI/2) / (members - 1)
  my_angle  = arc_share * (my_index - 1)
  jitter    = random(0 .. (PI/3) / (members - 1))

  # the arc is centred on the direction from the enemy toward the pack's home point
  base = heading_of( normalize(home_point - enemy.position) )
  heading = normalize_angle(base - (PI - PI/3)/2 + my_angle + jitter)

  point = enemy.position + completion_dist * direction(heading)
  IF that point is not on the navigation mesh
    point = enemy.position                       # give up the fan for this tick

FUNCTION is_finished() -> bool
  IF a timeout was set AND it has elapsed                       RETURN true
  IF horizontal distance to the enemy < completion_dist - 2     RETURN true
  IF horizontal distance to the point <= 2                      RETURN true
  RETURN false
```

**Notes** — three decisions carry the behaviour.

**The arc is anchored on the home direction, not on the creature's own approach.** Every member
computes its point relative to the line from the enemy toward the pack's home point, so the whole
pack ends up on the *home side* of the enemy — between the intruder and the territory. That is why
a pack does not surround the player evenly: it interposes itself.

**The spacing is by squad index, and the index comes from the squad's ordering against this
enemy** — assigned once when the first member entered the attack (see
[`group_state_attack_inline.h`](group_state_attack_inline.h.md)). The jitter added per member is
scaled by the same member count, so a larger pack gets proportionally smaller random spread and
the arc does not become noise.

**The fan degrades gracefully three ways**: no squad, no index within it, or a computed point off
the navigation mesh all fall back to "walk at the enemy". None of them fails the state.

The completion test's first clause is a *give-up* rather than an arrival: if the enemy has come
within the ring radius by itself, the state is done and the composite advances the ladder. Without
it the creature would keep backing away to maintain its ring.

The arc arithmetic divides by *members minus one*, so a squad with exactly one living member
divides by zero. Nothing guards it; reaching that case requires being indexed in a squad of one,
which the squad's own bookkeeping apparently prevents. A rebuild should guard.

## `CStateGroupSquadMoveToRadius` — the radial variant

**Contract** — identical in every respect except the point computation and the completion test.
Aims at the point on the ring lying on the line from the enemy through the creature's own current
position, with a 1-unit approach margin, and finishes on timeout or within 1 unit of the point. No
squad is consulted at all.

```text
FUNCTION compute_point()
  direction = normalize(self.position - enemy.position)
  point = enemy.position + completion_dist * direction
  IF that point is not on the navigation mesh
    point = enemy.position
```

**Notes** — the radial variant is used for the two *inner* rungs of the stalking ladder, and that
choice is deliberate. The fan exists to spread the pack out at the outermost ring; once members
are already spread, aiming each one straight in along its own radius preserves the spacing without
recomputing it. It also means the inner rungs work identically for a solitary creature, which is
why the generic creature states can use this class too.

Its approach margins are tighter than the fanned variant's — 1 unit instead of 2, in both the path
parameters and the completion test — because the inner rings are smaller and a 2-unit slop there
would be most of the ring.

Both variants recompute the point **every tick** rather than latching it on entry: the ring follows
the enemy. That is what turns "approach a point" into "maintain a distance", and it is the single
line that makes the stalking ladder work at all.
