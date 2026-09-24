# src/xrGame/ai/monsters/controller/controller_state_attack_moveout_inline.h

> Leave cover when the enemy is out of sight and creep to where it was, in two hops, glancing
> around on the way.

**Needs** — [`controller_state_attack_moveout.h`](controller_state_attack_moveout.h.md) · [`controller.h`](controller.h.md) · [`controller_animation.h`](controller_animation.h.md) · [`controller_direction.h`](controller_direction.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`controller_state_attack_moveout.h`](controller_state_attack_moveout.h.md)
**Tier floor** — T3: path request plus a timed look-direction policy

## Purpose

The complement to the controller's retreat: the creature has broken contact, and now has to
re-find its enemy without simply charging. It walks in stealth posture along a two-hop route —
first to the cell from which the creature itself last saw the enemy, then to the cell the enemy
itself occupied — and while walking it alternates its gaze between the enemy's last known
position and the most open direction near it.

**Currently unreachable in play**: no state manager registers it.

## State

```text
RECORD MoveOutState
  phase                : ENUM { to_my_last_sighting_cell, to_enemy_last_cell }
  target_position      : vector
  target_vertex        : int
  enemy_vertex         : int    # latched once, on entry
  look_point           : vector
  look_point_updated_at: int
  look_change_delay    : int    # 2s or 4s, chosen with the look point
```

The enemy's vertex is read **once, on entry**, directly from the enemy object rather than from
the creature's own memory of it. The source calls this "cheating here", and it is: the creature
gets a fact it has not perceived. A rebuild that wants honest perception must replace it with the
remembered enemy vertex and accept that the state sometimes has nothing to walk toward.

## `CStateControlMoveOut`

**Contract** — on entry prime the path builder, latch the enemy vertex and start in the first
phase. Each tick recompute the target, hand it to the path builder with a long rebuild interval,
request the stealth action with acceleration disabled, set the aggression sound, refresh the look
point on its own clock and request the stealth body pairing. Starts only when the enemy is *not*
currently visible; finishes when it becomes visible, when the creature is hit after entry, when
it gets within one unit of the remembered enemy position, or after ten seconds.

**Invariants** — the phase only advances; it never returns to the first hop. The look point is
always lifted 1.5 units above the ground, so the creature looks at eye height rather than at the
floor.

```text
FUNCTION update_target()
  IF phase == to_my_last_sighting_cell AND self.position ~= target_position (within 0.05)
    phase = to_enemy_last_cell

  IF phase == to_my_last_sighting_cell
    target_vertex = memory.my_vertex_when_enemy_last_seen
                    OR enemy_vertex IF that memory is absent
  ELSE
    target_vertex = enemy_vertex

  target_position = navigation.position_of(target_vertex)

FUNCTION update_look_point()
  IF last_hit_time > enemy_last_seen_time
    look_point = self.position + last_hit_direction * 5, raised 1.5
    stamp(); RETURN                                  # a fresh hit always wins, immediately

  IF look_point_updated_at + look_change_delay > now()  RETURN

  IF coin(LOOK_COVER_PROBABILITY) AND this is not the first refresh
    angle = navigation.most_open_low_cover_direction(self.vertex)
    look_point = self.position + direction(angle) * 3
    look_change_delay = BASE_DELAY                   # glance away: look again soon
  ELSE
    look_point = enemy_last_known_position
    look_change_delay = BASE_DELAY * 2               # look at the threat: hold it longer

  look_point.raise(1.5); stamp()
```

**Notes** — the two-hop route is the interesting decision. Walking straight to where the enemy
stood is wrong, because the creature may have been watching from a place with a clear line that
the direct route does not follow; walking first to *its own* last vantage point re-establishes
the line of sight the creature had, and only then does it advance. The fallback when that memory
is missing collapses the two hops into one.

The look policy encodes character rather than tactics: a 30 percent chance each refresh of
glancing at the most exposed direction nearby, and holding a look at the enemy twice as long as a
glance away. The "not the first refresh" guard forces the *first* look to be at the enemy, so the
creature never leaves cover already looking the wrong way.

The rebuild interval of 20 seconds effectively means "do not rebuild": the route is computed once
and followed. That is correct for this state, whose target is a fixed remembered cell, and wrong
for any state chasing a moving enemy.

The ten-second cap appears twice with the same value — once as a named constant and once written
inline in the finish predicate. Both must change together in a rebuild.
