# src/xrGame/script_animation_action_inline.h

> The setters, and the one decision they encode: naming a clip makes the action wait, naming a posture does not.

**Needs** — [`script_animation_action.h`](script_animation_action.h.md)
**Used by** — [`script_animation_action.h`](script_animation_action.h.md)
**Tier floor** — T3: field assignments

## Purpose

Construction and the two setters. Everything interesting is in which fields each setter
writes, because writing them is how the part's *kind* is chosen.

## `set_animation`

**Contract** — names the clip to play, selects the animation goal type, **clears the
completed flag** so the action waits for the clip, resets the mental state to the wary
default, and clears the movement-controller flag.

```text
FUNCTION set_animation(clip_name)
  animation    = clip_name
  goal_type    = ANIMATION
  completed    = false            # this part now has something to wait for
  mental_state = DANGER           # the wary default, imposed rather than preserved
  use_movement_controller = false # reset: a caller wanting it must set it AFTER this
```

**Invariants** — the mental state is overwritten, not left alone. A script that sets a
posture and then names a clip silently loses the posture. Since the goal type would have made
the posture irrelevant anyway, the effect is only visible if a rebuild keeps both fields live
— which is an argument for the tagged union the record wants to be.

## `set_mental_state`

**Contract** — names the posture, selects the mental goal type, **sets the completed flag**
so the action does not wait, and clears the movement-controller flag.

**Invariants** — the completed flag going *true* here is the whole reason the two setters
differ. Adopting a posture takes effect immediately and there is nothing to wait for; a
script action whose only animation part is a posture change ends on the update it starts.

## Construction

**Contract** — four forms, each delegating to a setter or writing the monster fields:

- **empty** — a completed mental-state part in the wary posture. Harmless to carry.
- **clip name**, optionally with the movement-controller flag — delegates to the animation
  setter and *then* applies the flag, which is the only correct order given that the setter
  clears it.
- **mental state** — delegates to the posture setter.
- **monster animation kind plus index** — writes the monster fields directly, clears the
  completed flag, and clears the movement controller. It is the only constructor that does
  not go through a setter, which is why it leaves the goal type and mental state at their
  zero values rather than their declared defaults.

## `initialize`

**Contract** — does nothing. Present so that every part of a script action has the same
shape and the action can initialize its parts uniformly without asking which kind each is.
