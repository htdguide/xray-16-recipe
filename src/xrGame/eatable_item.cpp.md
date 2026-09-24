# src/xrGame/eatable_item.cpp

> Consumable items: a use counter, the condition and booster effects one use applies, a weight that drops as the item is eaten, and the rule for when a spent item leaves the world.

**Needs** — [`eatable_item.h`](eatable_item.h.md) · [`physic_item.h`](physic_item.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Level.h`](Level.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`ui/UIActorMenu.h`](ui/UIActorMenu.h.md)
**Used by** — reached through its declarations in [`eatable_item.h`](eatable_item.h.md); callers name that, not this file.
**Tier floor** — T3: a counter and a table of effects read from configuration

## Purpose

Everything consumable in the game — a tin of food, a medkit, a bandage, an anti-radiation
drug, a bottle of vodka — is this one behaviour with different numbers. The numbers all live
in the item's configuration section; this file owns the counter, the weight curve, and the
question of when a used-up item stops existing.

It is a mixin rather than a class of its own: it is combined with the physical-item behaviour
in [`eatable_item_object.cpp`](eatable_item_object.cpp.md).

## State

```text
RECORD EatableItem EXTENDS InventoryItem
  physic_item     : PhysicItem          # the other half of the combined object
  max_uses        : int (8-bit) = 1     # 255 means INFINITE
  remaining_uses  : int (8-bit) = 1
  remove_after_use: bool = true         # does the item vanish when spent
  weight_full     : real                # the configured weight, captured at load
  weight_empty    : real = 0            # what it weighs with nothing left
```

**Invariant** — a maximum of **255** means infinite uses, not 255 uses. The decrement is
skipped entirely at that value, so such an item never empties. It is the only reserved value
in the counter and it costs the top of the range.

**Invariant** — `weight_full` is captured from the item's already-loaded weight rather than
read from its own key. So an item's ordinary weight key is its *full* weight and the empty
weight is a separate key. A rebuild that reads both from configuration must keep that
asymmetry or every consumable becomes lighter.

**Invariant** — an item with `remove_after_use` false stays in the inventory when spent. That
is how refillable and reusable containers work, and it is why "empty" and "useless" are two
different questions.

## `Load`

**Contract** — reads the four tunables from the item's section, all optional with defaults
(one use, removed after use, zero empty weight). Sets the remaining uses to the maximum —
a freshly configured item is full. Then, if the item uses the condition system, seeds its
condition from the use fraction.

**Notes** — the condition seed is computed as remaining divided by maximum in **integer**
arithmetic before being widened to a real. The result is therefore 1 when the item is full
and 0 in every other case, never anything between. Every one of the three places this
expression appears has the same defect. The visible consequence is that a partly used item
shows as condition zero. A rebuild should divide in real arithmetic; reproducing the integer
division reproduces a bug, not a behaviour.

## `UseBy`

**Contract** — consumes one use on behalf of a living entity. Requires that the entity owns
an inventory, that it is *this item's* inventory, and that the item's parent is that entity —
three assertions that together forbid using an item out of somebody else's pockets.

```text
FUNCTION UseBy(entity)
  influences = the medicine-influence values of this item's section
  entity.conditions.apply(influences, section)

  FOR EACH booster parameter kind
    IF the section names that booster THEN
      entity.conditions.apply_booster(that booster's values, section)

  IF multiplayer AND this is the server THEN
    broadcast a "player used a booster" event naming the entity and this item

  IF max_uses is not the infinite marker THEN
    remaining_uses = max(remaining_uses - 1, 0)

  IF the item uses the condition system THEN reseed the condition from the use fraction
  RETURN true
```

**Invariants** — the influences are the *immediate* effects (health, hunger, bleeding,
radiation, power) and the boosters are the *timed* ones. They are loaded from the same section
under different keys, applied in that order, and the booster loop is driven by the fixed
enumeration of booster kinds, so adding a booster kind means adding a key name.

**Invariants** — the effects are applied **before** the counter is decremented, and the
function cannot fail after the assertions. There is no path in which an item is consumed
without taking effect or takes effect without being consumed.

**Invariants** — in multiplayer the effect is applied on the server and broadcast as an
event; the clients do not compute it. Consumables are therefore authoritative on the server,
which matters because health is.

**Notes** — the returned value is always true. The signature promises a failure case the
implementation does not have.

## `Useful`

**Contract** — whether the item is worth existing. False once it is spent *and* configured to
be removed when spent; otherwise it defers to the inventory item's own answer. This is the
predicate the two hand-over hooks below consult.

## `OnH_B_Independent` · `OnH_A_Independent`

**Contract** — the two halves of "this item has left a container and become a free object in
the world", before and after the transition.

- **Before**: a useless item is made invisible and disabled, and its physical half is flagged
  ready for destruction. It must not be seen falling to the floor.
- **After**: a useless item destroys itself — but only when this process is authoritative for
  it (it is the local copy and this is the server).

**Invariants** — the split matters. Hiding must happen before the object is placed in the
world and destruction must happen after, because destroying an object mid-transition leaves
the inventory holding a dead reference. The ordering is the load-bearing part of this pair.

## `Weight`

**Contract** — linear interpolation between the empty and full weights by remaining uses.
Applies only when the item uses the condition system; otherwise the inherited weight stands.

```text
FUNCTION Weight() -> real
  IF NOT using_condition THEN RETURN inherited weight
  per_use = (weight_full - weight_empty) / max_uses     # zero when max_uses is zero
  RETURN weight_empty + remaining_uses * per_use
```

**Invariants** — a half-eaten tin weighs half. This is visible to the player through the
carry-weight limit and is the only reason the two weights are configured separately.

**Notes** — an item with the infinite-uses marker has a maximum of 255 and a remaining count
of 255, so it weighs its full weight forever, which is right.

## `save` · `load`

**Contract** — persists exactly one field, the remaining-use count, after the inventory
item's own state. Everything else is re-derived from configuration on load. That is the
correct cut: uses are the only thing about a consumable that the world changed.

## `net_Spawn`

**Contract** — after the inherited spawn, reseeds the condition from the use fraction — the
same expression, with the same integer-division defect, as in `Load`.

## `_construct`

**Contract** — resolves and caches the pointer to the physical half of the combined object.
It is a downcast of this object to its sibling base, which works because the two are always
combined; see [`eatable_item_object.h`](eatable_item_object.h.md). A rebuild that composes
rather than multiply-inherits passes the reference in instead.
