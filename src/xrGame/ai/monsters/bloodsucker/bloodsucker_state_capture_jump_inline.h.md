# src/xrGame/ai/monsters/bloodsucker/bloodsucker_state_capture_jump_inline.h

> Where the brain goes to get out of the way: while the scripted seize-and-leap owns the creature, its behaviour is one node that stands still and does nothing else.

**Needs** — [`bloodsucker_state_capture_jump.h`](bloodsucker_state_capture_jump.h.md) · [`state_custom_action.h`](../states/state_custom_action.h.md)
**Used by** — [`bloodsucker_state_capture_jump.h`](bloodsucker_state_capture_jump.h.md)
**Tier floor** — T3: a single held action

## Purpose

The drag jump is driven from outside the brain — an animation with a completion callback that ends in a scripted leap. Something still has to occupy the creature's behaviour slot for the two or three updates it takes, or the ordinary selector would re-issue movement requests over the animation and the seize would break.

So this state exists to be inert. Standing idle with no timeout is exactly the right answer: it cancels whatever the previous state was asking for and asks for nothing that conflicts.

## State

```text
SUBSTATES
  hold : generic "hold one action"
```

Stateless otherwise.

## `BloodsuckerCaptureJumpState`

**Contract** — every update, select the single child, run it, record it as previous. Fill it with: stand idle, no timeout, and no vocalisation.

```text
FUNCTION execute()
  select(hold)
  hold.execute()
  previous = hold
```

**Notes** — the vocalisation fields are present but disabled: a creature in the middle of a scripted grab should not be playing its ambient idle sound over it. That is the only decision in the file.

A note at the top of the update marks a check of alife control that was never written. Nothing depends on it: the state is only entered when the creature's own flags say a scripted sequence owns it, so the check would be redundant.
