# src/xrGame/ai/monsters/group_states/group_state_eat_inline.h

> The pack feeding sequence: walk to the corpse, drag it into cover if you can, play the
> tearing animation, eat, then withdraw and settle — with a twenty-second satiety clock governing
> the whole thing.

**Needs** — [`group_state_eat.h`](group_state_eat.h.md) · [`group_state_eat_drag.h`](group_state_eat_drag.h.md) · [`group_state_eat_eat.h`](group_state_eat_eat.h.md) · [`group_state_custom.h`](group_state_custom.h.md) · [`../states/state_move_to_point.h`](../states/state_move_to_point.h.md) · [`../states/state_hide_from_point.h`](../states/state_hide_from_point.h.md) · [`../states/state_custom_action.h`](../states/state_custom_action.h.md) · [`../states/state_data.h`](../states/state_data.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`group_state_eat.h`](group_state_eat.h.md)
**Tier floor** — T2: sequencing plus a physics capture whose release must be exact on every exit

## Purpose

Feeding is the longest-running creature behaviour in the game and the one with the most
externally visible side effects: it physically captures a ragdoll, it locks a shared resource
against other creatures, and it drives the numbered animation vocabulary. This composite owns that
sequence and, more importantly, owns its **unwinding** — which is why its two exit paths are the
largest functions in the file.

Compared with the generic (solitary) feeding state, the pack version adds the dragging rung, the
corpse locking, and the flavour-animation hand-offs.

## State

```text
RECORD GroupEatState
  corpse        : object   # the corpse this activation latched, at entry
  time_last_eat : int      # when feeding last ended; 0 means "never fed"
```

`TIME_NOT_HUNGRY` is 20 seconds, authored in code, and it appears twice: as the satiety interval
and as the amount by which the clock is pushed *forward* when a flavour animation interrupts.

The latched corpse and the creature's current corpse are compared rather than assumed equal —
that comparison is the composite's way of noticing that something else changed the creature's
claim underneath it.

## `reselect_state`

**Contract** — pick the next substate from the one that just finished. Called only when no substate
is active, which is how the base state machinery signals "the previous one completed".

```text
FUNCTION reselect_state()
  IF a flavour animation was requested
    select custom; clear the request
    time_last_eat = now() + TIME_NOT_HUNGRY     # an interrupting animation suppresses hunger
    RETURN

  IF the creature asked to resume eating (saved_state == eat)
    clear the request; release any physical capture; select eat; RETURN

  IF nothing ran yet                       -> approach at a walk
  IF approach did not complete             -> keep approaching
  IF approach completed
      IF we can drag AND the corpse is draggable -> drag
      ELSE IF eating will accept                 -> eat
      ELSE                                       -> approach again
  IF drag did not complete                 -> keep dragging
  IF drag completed
      IF eating will accept
         request flavour clip 15 (tear at a corpse); remember to resume eating; select custom
      ELSE                                       -> approach again
  IF eating just ended
      stamp time_last_eat
      -> withdraw IF no longer hungry, else approach again
  IF withdrawal ended                      -> rest
  IF resting ended                         -> rest again
  otherwise                                -> rest
```

**Notes** — three decisions here are not visible from the shape.

**The approach rung is a loop, not a step.** If the eating state refuses (the creature is not close
enough to the nearest bone of the ragdoll), the sequence re-enters the approach rather than
failing. Since the corpse may have been dragged, rolled or settled since the last attempt, this
converges; the approach's completion distance and the eating state's acceptance distance differ by
half a unit, which is what guarantees it terminates rather than cycling.

**The tearing animation is inserted between dragging and eating**, not merged into eating, and it
is routed through the numbered-animation machine with an explicit "resume eating afterwards"
note. That note is stored *on the creature*, not in this state, because the animation's completion
runs the brain from the top (see [`../dog/dog.cpp`](../dog/dog.cpp.md)) and the sequence has to be
re-entered from outside.

**A flavour animation pushes the satiety clock forward by a full satiety interval.** The creature
that stops to growl is treated as having just eaten. That is what prevents a creature from
immediately resuming a meal it was interrupted out of.

Two commented-out rungs remain — a running approach and a corpse-inspection step — and the running
approach substate is still registered and parameterised even though no path selects it. A rebuild
can drop it.

## `setup_substates`

**Contract** — fill the parameter record for whichever substate just became active. Five are
parameterised: the two approaches, the corpse inspection, the withdrawal and the resting pause.

The parameters that carry decisions:

- **Both approaches aim at the nearest *bone* of the ragdoll**, not at the corpse's origin — or at
  the origin when the corpse has no active physics shell. A settled ragdoll's origin can be
  metres from its body, and a creature that walks to the origin stands beside the meal.
- Both stop at the section's *distance to corpse*, accelerate calmly with braking, and play the
  idle sound at the section's idle delay. The only difference between them is the gait.
- **The withdrawal runs 15 units from the corpse and searches for cover between 20 and 30 units
  away with a 25-unit search radius.** Those three numbers are authored in code and are
  independent of the retreat distance, which is why a creature that has eaten walks further than
  it retreats.
- The inspection and the resting pause are both half-second holds in standing idle; they differ
  only in which sound plays.

## `check_start_conditions`

**Contract** — true immediately if the creature already holds a claimed corpse. Otherwise the
conjunction of four facts: a corpse is remembered, it lies **inside the creature's home region**,
the creature is hungry, and no other creature has locked it.

**Notes** — the home-region test is what keeps a pack from wandering across the level to a kill it
heard about. The lock test is what turns a pack feeding into a queue; see
[`../dog/dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md) for the claiming side.

## `check_completion` / `hungry`

**Contract** — the composite finishes when the latched corpse is no longer the creature's corpse,
or when the creature has fed within the last satiety interval. Hunger is simply "never fed, or fed
more than twenty seconds ago".

## `finalize` / `critical_finalize`

**Contract** — the unwinding, and the most defect-prone part of the file. Both paths must: restore
the corpse's use timeout from the creature's section, unlock the corpse for other creatures,
release any physical capture, clear the creature's claim, and raise the "just finished eating"
flag that the resting state reads.

**Invariants** — the corpse's guards are restored **only when the latched corpse is still the
creature's corpse**; otherwise some other activation owns it and must not have its lock cleared.

**Notes** — the forced-exit path is visibly inconsistent with itself: it performs the guarded
restore, then performs a second unguarded release when an enemy exists, and then clears the claim
and raises the flag *unconditionally* — outside every guard, including the one that was supposed
to protect another activation's corpse. The guarded block's condition additionally includes the
composite's own completion test, so a forced exit while the creature is *still hungry* skips the
timeout and lock restoration entirely and leaves the corpse locked against every other creature
until something else clears it.

This is the single least defensible piece of code in the chapter and a rebuild should not
reproduce it. The correct unwinding is the clean path's: guard on ownership, restore timeout and
lock, release the capture, clear the claim, raise the flag — once, on both paths. The observable
difference is a corpse that other creatures refuse to approach after a pack is scattered mid-meal,
which is a known behaviour of the original.

The clean path also raises the "just finished eating" flag, which
[`group_state_rest_idle_inline.h`](group_state_rest_idle_inline.h.md) consumes to send the creature
to the *inner* home ring and sit down rather than resume wandering. That is the one piece of state
that crosses between the feeding and resting behaviours.
