# src/xrGame/ai/monsters/controller/controller_state_attack_hide_inline.h

> The controller breaks contact: pick a covered navigation vertex away from the enemy, run to it,
> and switch the creature's outward demeanour on the way.

**Needs** — [`controller_state_attack_hide.h`](controller_state_attack_hide.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`controller.h`](controller.h.md) · [`controller_animation.h`](controller_animation.h.md) · [`controller_direction.h`](controller_direction.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`controller_state_attack_hide.h`](controller_state_attack_hide.h.md)
**Tier floor** — T3: cover query plus path request; the navigation graph is somebody else's chapter

## Purpose

The controller is a ranged, fragile creature: its whole tactic is to be somewhere the enemy is
not looking. This state is the "get there" half of that tactic. It asks the cover system for a
vertex that is well covered *from the enemy's position* within an annulus around the creature,
then hands that vertex to the path builder and runs.

It is the only controller-specific leaf state the shipped brain actually registers, and it is
registered under the *custom* slot — meaning the automatic selector never picks it, and it is
reached only when a script forces the creature into that state. See
[`controller_state_manager.cpp`](controller_state_manager.cpp.md).

## State

```text
RECORD ControlHideState
  target_position : vector       # where we are running
  target_vertex   : int          # the navigation vertex of that position
  cover_reached   : bool         # set false on entry; never read afterwards
  fast_run        : bool         # this activation started more than 20 units out
  time_finished   : int          # declared, never written
```

Two of the five fields are vestigial. `cover_reached` is initialised and never consulted, and the
finish stamp is declared and never assigned; the arrival test reads the navigation state directly
instead. A rebuild should carry only the target and the sprint flag.

## `CStateControlHide`

**Contract** — on entry, choose a target and prime the path builder. Each tick, re-assert the
movement request (run), the path parameters, the aggressive acceleration profile, the aggression
sound, and a head-look direction; also drop out of the sprint posture once close enough. On exit,
by either path, put the creature back into its *danger* demeanour. Finishes when the creature
stands on the target vertex and the path builder reports it is no longer moving. Always willing
to start.

**Invariants** — the demeanour is restored on both exits. The path is rebuilt every tick (rebuild
interval zero) and stops exactly at the target (end distance zero) — this state does not want a
smoothed approach, it wants the specific covered cell.

```text
FUNCTION select_target()
  point = cover_system.find_cover(from = enemy_position,
                                  min_radius = 10, max_radius = 30)
  IF point EXISTS
    target_position, target_vertex = point.position, point.vertex
  ELSE
    target_vertex   = 0                       # fall back to vertex zero of the level mesh
    target_position = navigation.position_of(target_vertex)

  fast_run = distance(target_position, self.position) > 20
  IF fast_run AND coin_flip(50 percent)
    set_demeanour(idle)                       # half the time, sprint without looking alarmed

FUNCTION execute()
  IF fast_run AND distance(target_position, self.position) < 5
    fast_run = false
    set_demeanour(danger)

  request_action(run)
  path.target      = (target_position, target_vertex)
  path.rebuild_every = 0          # every tick
  path.stop_distance = 0          # arrive exactly
  path.use_covers    = false      # the destination is already the cover decision

  animation.accelerate(aggressive, braking = false)
  sound.play(aggressive, delay = section.attack_sound_delay)

  IF last_hit_time > enemy_last_seen_time
    look_at(self.position + last_hit_direction * 5, raised by 1.5)
  ELSE
    look_at(enemy_position)

  set_body_pairing(torso = run, legs = run)

FUNCTION is_finished() -> bool
  RETURN self.vertex == target_vertex AND NOT path.is_moving
```

**Notes** — the fallback when no cover is found is *vertex zero of the level mesh*, an arbitrary
corner of the map, not "stay put". That is a bug-shaped decision with a real consequence: a
controller that cannot find cover walks to the origin of the navigation mesh. A rebuild should
decide deliberately whether to keep it; the shipped levels rarely trigger it because the annulus
is generous.

The head-look rule encodes a small piece of character: if the creature has been shot more
recently than it has seen its enemy, it looks *at the shot*, at roughly chest height above the
floor, rather than at the stale remembered enemy position. The 1.5-unit lift is the same
"where a person's torso is" offset used elsewhere in the monster states.

The coin flip on entering a long sprint is the only randomness here, and it is deliberately
cosmetic: half of long retreats are played with the calm posture, which reads as the creature
withdrawing rather than fleeing.
