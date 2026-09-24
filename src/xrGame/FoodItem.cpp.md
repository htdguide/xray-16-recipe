# src/xrGame/FoodItem.cpp

> Food and drink as a class identifier: the consumable item behaviour with no additions.

**Needs** — [`FoodItem.h`](FoodItem.h.md) · [`eatable_item_object.h`](eatable_item_object.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a class identity and nothing else

## Purpose

The class the shipped spawn data names for bread, sausage, vodka, energy drinks and the
rest. All of the behaviour — being carried, being used, applying its configured deltas to
the consumer's condition, decrementing its use count and deleting itself when spent —
belongs to the consumable item base. This class supplies only the identity the data asks
for.

Note the split that does exist: medical items are a *different* class identifier with the
same base, because the inventory groups them separately and the use key differs. Food and
medicine are otherwise indistinguishable in code.

## State

`Stateless.`

## `CFoodItem`

**Contract** — construction and destruction, both empty; every operation is the consumable
item base's.
