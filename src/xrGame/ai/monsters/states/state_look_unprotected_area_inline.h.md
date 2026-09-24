# src/xrGame/ai/monsters/states/state_look_unprotected_area_inline.h

> Implements "face where I am most exposed", by asking the level's per-vertex cover data which direction offers the least protection and turning that way.

**Needs** — [`state_look_unprotected_area.h`](state_look_unprotected_area.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`state_look_unprotected_area.h`](state_look_unprotected_area.h.md)
**Tier floor** — T3: one cover query and a facing request

## Purpose

A panicking creature that cannot flee should look like it is watching the direction it is
in danger from. The engine does not track *where the danger is* precisely enough for that,
so this state substitutes a proxy that is always available: the level's authored per-vertex
cover values. The direction with the *least* high cover is the direction the creature is
most exposed from — and, equally, the way out. The creature faces it.

The whole decision is made once, at entry, and never revised: a panicking creature does not
re-evaluate. That is deliberate — re-querying per tick would make a cornered creature's
head swivel as it shuffled between cells.

## `CStateMonsterLookToUnprotectedArea`

**Contract** — on entry, queries the creature's current navigation vertex for the heading
whose high-cover value is smallest over a 30-degree window, then places a target point one
metre away along the *opposite* heading, and stores it. Each tick it asserts the action and
modifiers, requests a facing toward that stored point, and optionally plays a sound.
Completion is the two-mode rule: timeout if one was given, otherwise the moment the turn
finishes.

```text
FUNCTION initialize()
  # the cover query works from the navigation vertex, not the exact position
  angle = level_graph.vertex_high_cover_angle(
              object.location.level_vertex_id,
              window = 30 degrees,
              select = minimum)          # "least covered" heading

  direction = unit vector at heading (angle + 180 degrees)
  target_point = object.position + direction * 1 metre

FUNCTION execute()
  object.animation.action = data.action
  object.animation.set_modifiers(data.spec_params)
  object.direction.face_target(target_point)
  IF data.sound_type IS PRESENT THEN play it, honouring data.sound_delay

FUNCTION check_completion() -> bool
  IF data.time_out != 0 THEN RETURN time_state_started + data.time_out < now()
  RETURN NOT object.control.direction.is_turning()
```

**Invariants** — the target point is only one metre out, so it is a *direction* expressed
as a position. A rebuild storing a heading directly is equivalent and clearer; the position
form exists only because the facing request takes a point.

## Notes

**The half-turn.** The query returns the least-covered heading and the state adds 180
degrees to it. Read literally that faces the creature *away* from its most exposed side —
toward the cover behind it. Whether the sign is a bug or whether the underlying query's
selector already returns the reverse heading is not recoverable from the source; the
observable behaviour of a panicking creature is what a rebuilder must match, and that must
be checked against a running original.

**A dead lift.** Entry computes the creature's position raised by 30 centimetres and then
never uses it. A leftover from a version that raycast from eye height; it decides nothing.
