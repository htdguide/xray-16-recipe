# src/xrGame/object_actions.h

> Declares the eighteen planner operators through which a creature draws, stows, aims, reloads, fires, throws and drops what it is holding — implemented in [`object_actions.cpp`](object_actions.cpp.md).

**Needs** — [`action_base.h`](action_base.h.md) · [`object_actions_inline.h`](object_actions_inline.h.md) · [`object_handler_space.h`](object_handler_space.h.md)
**Used by** — [`object_actions.cpp`](object_actions.cpp.md) · [`object_actions_inline.h`](object_actions_inline.h.md) · [`object_handler_planner.cpp`](object_handler_planner.cpp.md) · [`object_handler_planner_missile.cpp`](object_handler_planner_missile.cpp.md) · [`object_handler_planner_weapon.cpp`](object_handler_planner_weapon.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the concrete operators of the object-handling planner. Each is a small class whose
three lifecycle points — set up, run a frame, tear down — issue inventory commands and
report world-property changes back to the planner. Substance is split between
[`object_actions.cpp`](object_actions.cpp.md) (the concrete operators) and
[`object_actions_inline.h`](object_actions_inline.h.md) (the two bases, which are
templates).

Exported units:

- `CObjectActionBase` — the base every operator derives from. Holds the item and the
  planner's property storage, clears the two aim-completed properties on setup, and offers
  two helpers for interrupting a stow animation in progress.
- `CObjectActionMember` — a base that additionally sets one named property to one value the
  moment the operator completes. Most "and now this is true" operators need nothing more.
- `CObjectActionCommand` — issue one raw inventory command and be done.
- `CObjectActionShow` / `CObjectActionHide` — bring the item into hand, or put it away.
- `CObjectActionStrapping`, `CObjectActionStrappingToIdle`, `CObjectActionUnstrapping`,
  `CObjectActionUnstrappingToIdle` — the four legs of the sling-over-shoulder transition,
  each driven to completion by an animation-end callback rather than by a timer.
- `CObjectActionReload` — reload, with the checks that stop a creature reloading forever.
- `CObjectActionFire` / `CObjectActionFireNoReload` — hold the trigger; the second stops
  after one burst.
- `CObjectActionQueueWait` — wait out a burst that has already been fired.
- `CObjectActionAim` — aim, and release the burst-stop latch once aimed.
- `CObjectActionSwitch` — operate the firing-mode toggle.
- `CObjectActionDrop` — relinquish ownership of the item to the world.
- `CObjectActionIdle` / `CObjectActionIdleMissile` — stand holding it, for ordinary items
  and for thrown ones.
- `CObjectActionThrowMissile` — throw it, with a hold time scaled to the target's distance.
