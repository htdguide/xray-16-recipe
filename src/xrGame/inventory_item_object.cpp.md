# src/xrGame/inventory_item_object.cpp

> The join between "a thing that can be carried" and "a thing that exists physically in the world" — and the order in which the two halves are told about every event.

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`physic_item.h`](physic_item.h.md)
**Used by** — reached through its declarations in [`inventory_item_object.h`](inventory_item_object.h.md); callers name that, not this file.
**Tier floor** — T2: dispatch ordering across two behaviour mix-ins

## Purpose

Every method here forwards to the same method on both halves. The file looks like
boilerplate and is not: **the order of the two forwards is the content**, and it is
different for different events. Getting one backwards produces a bug that only shows on the
frame an item changes hands.

A rebuild that composes behaviours differently — components on an entity, traits, an
explicit list of listeners — still has to decide these orderings, and this page is the list
of them.

## State

`Stateless.` All state belongs to the two halves.

## `_construct`

**Contract** — runs both halves' own construction, then reports the joined object. Nothing
has been loaded from configuration yet; this is the phase where the halves may only set up
their own defaults. Called by the class-identifier factory before the spawn record is
applied.

## `Load` / `reload`

**Contract** — reads the item's configuration section. **Physical half first, carryable half
second.** The carryable half's load reads values the physical half has already established —
notably the visual model, from which the grid footprint and attachment bones are derived —
so the reverse order reads them unset.

## `reinit`

**Contract** — resets to a just-spawned state without re-reading configuration, run when an
item is re-used or a level restarts. **Carryable half first**, the reverse of `Load`,
because the carryable half's reset releases claims on the physical half's body and those
claims must be gone before the body itself is reset.

## The four owner-change edges

**Contract** — an item is either *independent* (lying in the world, physically simulated,
registered with the collision database) or a *child* (carried by someone, its body disabled,
its position derived from a bone of its holder). Each direction has a before and an after
hook, and the pairing of the two halves inverts between them.

```text
becoming independent   before : carryable half, then physical half
                       after  : carryable half, then physical half
becoming a child       before : physical half, then carryable half
                       after  : physical half, then carryable half
```

**Notes** — the rule under the table is **the half that is losing responsibility speaks
first**. Going independent, the carryable half is handing the object back to the world, so
it releases first and the physical half then wakes the body. Becoming a child, the physical
half is putting the body to sleep and detaching it from the world, and only once that is
done may the carryable half attach the item to its new holder's skeleton — otherwise an
active body and a bone attachment both drive the same transform for one frame, and the item
visibly snaps.

The `just_before_destroy` flag on the independent-before edge distinguishes "you are being
dropped" from "you are being deleted while held", which must not spawn a live body at all.

## `Hit` / `UpdateCL` / `OnEvent` / `renderable_Render` / `save` / `load`

**Contract** — each forwards to both halves, **physical first**. Damage, the per-frame
update, a network event, drawing and persistence all want the body's state settled before
the carryable half reacts to it: damage is applied to the body and then converted into
condition loss, the frame update moves the body and then the carryable half reads the new
position, and the saved record writes the body's pose before the item's own fields so the
restore can reconstruct in the same order.

## `net_Spawn`

**Contract** — brings the object to life from its server record. Three steps in a fixed
order, and the first of them is the one that must not move.

```text
FUNCTION net_Spawn(server_record) -> bool
  install_upgrades_from(server_record)   # before anything reads a tuned number
  ok = physical_half.net_Spawn(server_record)
  carryable_half.net_Spawn(server_record)
  RETURN ok
```

**Notes** — upgrades rewrite the item's cost, weight, immunities and handling numbers (see
[`inventory_item_upgrade.cpp`](inventory_item_upgrade.cpp.md)). They are applied *first*
because both halves' spawn reads those numbers — the physical half sizes the body from the
weight, the carryable half caches the cost and the handling factor. Applying them afterwards
would leave a live object built from the un-upgraded values.

Only the physical half's result decides success. The carryable half has no failure mode:
an item that cannot be carried is still an object in the world.

## `net_Destroy`

**Contract** — **carryable half first**, the mirror of spawn. The carryable half's teardown
removes the item from whatever inventory holds it and from the scheduler; only then may the
physical half remove the body from the physics world. The strict teardown order the system
requirements assert at runtime — nothing referenced by the scheduler, the render graph or
the physics world when memory is released — is enforced here by this pairing.

## `net_Import` / `net_Export`

**Contract** — only the carryable half is on the wire. The physical half's state reaches the
network through the item's own quantized position and orientation, which the carryable half
writes; re-sending the body's full description would be redundant and does not fit the
per-entity message budget.

## `activate_physic_shell` / `on_activate_physic_shell`

**Contract** — two names for one event, split so it can be intercepted. The outer call goes
to the carryable half, which decides *whether* a body should be created at all and with what
initial impulse; if it decides yes, it calls back into the inner one, which is where the
physical half actually creates it. A subclass that must veto or alter a body's creation
overrides the outer.

## `make_Interpolation` / the prediction hooks

**Contract** — only the carryable half participates in client-side prediction. The four
hooks bracket the physics step: capture before correction, blend between correction and
prediction, settle after, plus a development-build consistency check. A carried item's
position is its holder's, and a dropped item's is server-authoritative, so there is nothing
for the physical half to predict independently.

## `modify_holder_params` / `NeedToDestroyObject` / `Useful` / `ef_weapon_type`

**Contract** — `modify_holder_params` adjusts the holder's view range and field of view
while this item is in the hands, and is the carryable half's alone. `NeedToDestroyObject`
and `Useful` likewise delegate: the first is the lifetime question the level asks when
sweeping expired items, the second is whether a creature's evaluator should want this.
`ef_weapon_type` reports zero — the coarse weapon-class number that creature evaluators
compare, with zero meaning "not a weapon"; every weapon subclass overrides it.
