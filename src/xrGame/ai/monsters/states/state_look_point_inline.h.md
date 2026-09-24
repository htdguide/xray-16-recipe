# src/xrGame/ai/monsters/states/state_look_point_inline.h

> Implements turn-to-face, whose real decision is the two ways it can end: on a clock, or on the turn actually completing.

**Needs** — [`state_look_point.h`](state_look_point.h.md) · [`state_data.h`](state_data.h.md)
**Used by** — [`state_look_point.h`](state_look_point.h.md)
**Tier floor** — T3: a facing request and a completion test

## Purpose

Turning is the one creature action whose natural duration is decided by the *world* (how
far the head has to travel) rather than by the animation or by the caller. This state
exposes both readings, selected by whether the caller supplied a timeout, and that choice
is what callers use it for: a sweep of the surroundings wants "turn and stop", a menacing
stare wants "face him for three seconds".

## `CStateMonsterLookToPoint`

**Contract** — each tick it writes the action directly onto the animation component,
applies the animation modifiers, and issues a facing request toward the point with the
caller's *face delay* — a hold time during which the turn request may not be revised, which
is what keeps a creature from jittering between two nearly-equal targets. It optionally
starts a sound. Completion: if a timeout was given, expire on it; otherwise finish the
moment the direction component reports it is no longer turning.

```text
FUNCTION execute()
  object.animation.action = data.action.action     # written directly, not through set_action
  object.animation.set_modifiers(data.action.spec_params)
  object.direction.face_target(data.point, data.face_delay)

  IF data.action.sound_type IS PRESENT
    IF data.action.sound_delay IS PRESENT
      object.sound.play(data.action.sound_type, delay = data.action.sound_delay)
    ELSE
      object.sound.play(data.action.sound_type)

FUNCTION check_completion() -> bool
  IF data.action.time_out != 0
    RETURN time_state_started + data.action.time_out < now()
  RETURN NOT object.control.direction.is_turning()
```

**Invariants** — the turn-finished branch is only reachable in the tick *after* the first
execution, because the direction component is not yet turning when the state is entered.
Every caller of this state relies on the completion test being run after at least one
execution, which is the ordering the state framework guarantees.

**Notes** — this state assigns the action onto the animation component directly rather than
going through the creature's own action setter, which some sibling states use. The
difference is whether per-creature action-substitution rules (damaged variants of walk and
run, for example) are applied. Since this state only ever stands and turns, the two paths
agree today; a rebuild should pick one.
