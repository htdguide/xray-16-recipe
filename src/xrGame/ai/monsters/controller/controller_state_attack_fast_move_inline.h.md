# src/xrGame/ai/monsters/controller/controller_state_attack_fast_move_inline.h

> An unfinished sprint state: it runs, and it drops the creature's alarmed demeanour while it does. It never picks a destination, and nothing registers it.

**Needs** — [`controller_state_attack_fast_move.h`](controller_state_attack_fast_move.h.md) · [`controller.h`](controller.h.md) · [`state.h`](../state.h.md)
**Used by** — [`controller_state_attack_fast_move.h`](controller_state_attack_fast_move.h.md)
**Tier floor** — T3: one movement request and a demeanour switch

## Purpose

A fragment. The behaviour it carries is real and complete — while sprinting, the controller wears its *idle* outward demeanour rather than its *danger* one, and gets the danger demeanour back on every exit — but the movement half is a comment saying "select another cover" where the destination logic should be. Without a destination the state runs on whatever route the previous state left in the path builder.

The recipe keeps it because the demeanour rule is a real decision and because the file is the clearest surviving evidence that the controller's retreat was once meant to be a sequence of sprints between cover points rather than the single run it is now. What shipped instead is [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md), which folds the sprint and its half-the-time demeanour switch into the run-to-cover state.

## State

Stateless.

## `ControllerFastMoveState`

**Contract** — on entry, put the creature into its idle demeanour. Each update, request a run and nothing else. On either exit, restore the danger demeanour.

**Invariants** — the demeanour is restored on both exit paths, so no route out leaves the creature permanently wearing the wrong posture.

```text
FUNCTION initialize()   set_demeanour(idle)
FUNCTION execute()      request_action(run)        # no destination is chosen
FUNCTION finalize()     set_demeanour(danger)      # both exits
```

**Notes** — the demeanour is presentation only: it selects which animation set and which head-and-torso carriage the creature uses, not what it perceives or how it moves. Dropping it while sprinting is what makes a fleeing controller read as *leaving* rather than as *panicking*, and it is the same judgement the shipped retreat state makes on a coin flip.
