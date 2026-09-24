# src/xrGame/ai/monsters/controlled_entity_inline.h

> The generic implementation of being enthralled: save the allegiance, adopt the controller's, and restore it on every exit path.

**Needs** — [`controlled_entity.h`](controlled_entity.h.md) · [`controller/controller.h`](controller/controller.h.md)
**Used by** — [`controlled_entity.h`](controlled_entity.h.md)
**Tier floor** — T3: field writes and one call out to the controller

## Purpose

The body of the mixin declared in [`controlled_entity.h`](controlled_entity.h.md). Every
operation is three lines or fewer, and the whole file exists to get one invariant right:
the thrall's allegiance is restored on every path out of the hold.

## State

See [`controlled_entity.h`](controlled_entity.h.md).

## `set_under_control`

**Contract** — begin the hold. Records the controller, saves the thrall's team, squad and
group, and changes all three to the controller's.

```text
FUNCTION set_under_control(controller)
  self.controller <- controller
  saved_ids       <- (my team, my squad, my group)
  change my team, squad and group to the controller's
```

**Notes** — the allegiance change is the entire takeover. No brain is replaced, no plan is
injected; the creature simply becomes a member of the controller's group, and the game's
relation rules do the rest.

## `free_from_control`

**Contract** — end the hold normally. Restores the saved allegiance and clears the
controller.

## `on_die`

**Contract** — the thrall died while held. Tells the controller it has lost this thrall and
clears the handle. Does **not** restore the allegiance, and does nothing at all if the
thrall was not held.

## `on_destroy`

**Contract** — the thrall is being destroyed while held. Restores the allegiance, tells the
controller, and clears the handle. Does nothing if the thrall was not held.

**Notes** — the difference from death is the restore. A destroyed entity may have a server
record that survives the client object and is re-created later, and it must come back with
its own allegiance; a corpse has no allegiance worth restoring.

Neither path removes the thrall from the controller's list directly — both call into the
controller, which removes it by swapping the last entry into the vacated slot. So the
controller's thrall list has no stable order, and nothing may depend on one.

## `set_task_follow` / `set_task_attack`

**Contract** — set the task kind and the object it concerns. The thrall's own state layer
reads the pair each tick.

## `on_reinit`

**Contract** — clear the task's object and the controller handle at spawn. Note it does not
clear the task *kind*, which therefore survives a respawn as whatever it was. Nothing reads
the kind while the controller is empty, so it has never mattered.
