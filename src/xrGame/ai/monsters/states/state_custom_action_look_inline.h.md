# src/xrGame/ai/monsters/states/state_custom_action_look_inline.h

> Implements the leaf state "stand still, play this action, face this point, say this" — the simplest thing a creature can be told to do that still has a direction.

**Needs** — [`state_data.h`](state_data.h.md) · [`state.h`](../state.h.md)
**Used by** — [`state_custom_action_look.h`](state_custom_action_look.h.md)
**Tier floor** — T3: three setter calls and a clock comparison

## Purpose

The leaf state used whenever a behaviour wants the creature *stationary but oriented*:
looking around after hearing something, turning toward a call for help. It differs from
the plain stand-and-act state by one field — the point to face — and from the turn-to-point
state by not ending when the turn completes.

The declaration lives in a sibling header and the substance is here because the class is a
template; a rebuild with generics will merge the two.

## `CStateMonsterCustomActionLook`

**Contract** — on every behaviour tick it re-asserts three things on the creature: the
action (which selects an animation), the animation's modifier bits, and a request to face
the configured point. It optionally starts a sound, choosing between the caller's delay and
the creature's own configured delay for that sound kind. It never moves the creature and
never touches the path builder. It completes only on the configured timeout; with no
timeout it runs until the composite state above it chooses something else.

**Invariants** — re-asserting the same action and facing every tick is deliberate, not
wasteful: other components (a hit reaction, a squad order) may seize the animation or
direction channel between ticks, and this is how the state takes them back.

```text
FUNCTION execute()
  object.set_action(data.action)
  object.animation.set_modifiers(data.spec_params)
  object.direction.face_target(data.point)

  IF data.sound_type IS PRESENT
    IF data.sound_delay IS PRESENT
      object.sound.play(data.sound_type, delay = data.sound_delay)
    ELSE
      object.sound.play(data.sound_type)      # creature's own default repeat delay

FUNCTION check_completion() -> bool
  IF data.time_out == 0 THEN RETURN false     # 0 means "run until told otherwise"
  RETURN time_state_started + data.time_out < now()
```

**Notes** — this state reads the timeout from the shared action record's *top level*,
unlike its siblings which read it from a nested action field. That is a consequence of it
extending the action record rather than containing one, and it is the sort of
inconsistency a rebuild should flatten.
