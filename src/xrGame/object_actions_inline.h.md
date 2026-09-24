# src/xrGame/object_actions_inline.h

> The two bases every object-handling operator derives from: what they clear on setup, and how an operator declares its own completion.

**Needs** — [`object_actions.h`](object_actions.h.md) · [`object_handler.h`](object_handler.h.md) · [`object_handler_space.h`](object_handler_space.h.md)
**Used by** — [`object_actions.cpp`](object_actions.cpp.md) · [`object_actions.h`](object_actions.h.md)
**Tier floor** — T3: the operator base contract

## Purpose

These two bases are templates over the item type, which in the original language forces
their bodies into a header; the split is incidental, but the content is not — this is where
every object-handling operator's shared behaviour lives.

## State

```text
RECORD ObjectActionBase          # extends the generic planner action over a stalker
  item     : reference to the inventory item this operator acts on
  storage  : reference to the planner's world-property storage

RECORD ObjectActionMember        # extends the above
  property : WorldProperty       # set on completion
  value    : bool
```

## `CObjectActionBase`

**Contract** — construction binds the item, the owning creature, the property storage and a
name used for diagnostics. `set_property` writes one property into the planner's storage.
`object` hands out the owning creature.

**Contract** — `initialize` runs the generic action setup and then clears **both**
aim-completed properties, for both weapon slots.

**Invariants** — clearing the aim properties on *every* operator's setup, not just the
aiming ones, is the load-bearing decision on this page. Any operator starting means the
creature's hands are doing something other than holding a finished aim, so an aim that was
completed earlier is no longer valid. A rebuild that clears them only in the aim operators
will produce creatures that fire accurately out of a stow animation.

Both slots are cleared regardless of which slot the item is in, because a creature has one
pair of hands: starting anything invalidates the aim on either weapon.

## `stop_hiding_operation_if_any`

**Contract** — if the item currently in hand is not already hidden, abort whatever animation
it is playing *without firing that animation's completion callback*, and force both its
current and its next state to idle.

**Invariants** — suppressing the callback is the point. A stow animation's completion
callback would report "the item is now hidden" into the planner's world state, and the
whole reason for interrupting it is that the item is *not* going to be hidden — the plan
changed. Firing the callback would corrupt the world state the planner is mid-search over.

Setting both the current and the next state is likewise necessary: setting only the current
one lets the pending next state re-enter the interrupted animation on the following frame.

**Notes** — this is called by all four sling-transition operators at setup, because those
are the operators the planner interleaves with stowing most often.

## `prevent_weapon_state_switch_ugly`

**Contract** — does nothing. Its entire body is commented out.

**Notes** — the name and the dead body together record what it was for: forcing the held
item to idle and re-selecting the active slot, every frame, during the sling transitions,
to stop the item's own state machine from taking the item somewhere else mid-animation.
The four sling operators still call it every frame. Whatever problem it patched was either
fixed elsewhere or is being lived with. A rebuild should leave the call sites out and see
whether the problem returns, rather than reimplementing a workaround whose cause is not
recorded.

## `CObjectActionMember`

**Contract** — extends the base with one property and one value, and overrides the per-frame
step: run the base step, then — **only if the operator has completed** — write that property.

```text
FUNCTION execute()
  base.execute()
  IF completed THEN set_property(property, value)
```

**Invariants** — the property is written on completion and never on setup. An operator that
announced its effect before achieving it would let the planner believe a goal was reached
and move on, leaving the creature mid-animation with a world state that lies.

**Notes** — this base is what lets most operators be a constructor and nothing else: the
planner already knows the operator's effect from its declaration, and this class makes the
world state agree with it at exactly the right moment. Only operators needing more than one
effect, or an effect at a different time, override further.
