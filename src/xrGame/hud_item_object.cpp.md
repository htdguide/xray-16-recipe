# src/xrGame/hud_item_object.cpp

> Joins the inventory-item role and the held-item role into one object, and fixes the order in which the two halves see every event.

**Needs** — [`hud_item_object.h`](hud_item_object.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`HudItem.h`](HudItem.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: sequencing two role implementations

## Purpose

Every weapon in the game is two things at once: a record in an inventory grid with a weight,
a condition and a network identity, and a model animating in front of the camera with a state
machine, a sound set and an animation set. The two were written separately and this file is
the whole of their integration.

There is almost no logic here. What there *is*, and what a rebuild must copy exactly, is the
**order** in which the two halves are told about each event. The order is not uniform — it
flips depending on the event — and every flip is a real dependency.

## State

`Stateless.` Both halves own their own.

## The ordering rule

Stated once, because it explains every method below:

> **Coming into the world or into a hand, the inventory half goes first. Leaving, the held
> half goes first.**

The inventory half establishes existence — identity, section, weight, the network record —
and the held half needs all of that to pick its model and animations. Teardown must run the
other way, because the held half's animations and sounds reference the inventory half's
loaded data.

The one exception is attachment to an owner, where the held half goes first; see below.

## `_construct` and `Load`

**Contract** — construct and load both halves, inventory first. Loading reads the item's
configuration section twice, once per half, each taking the keys it owns.

**Notes** — the two halves read disjoint key sets from one section, which is why a single
section can describe both roles. A rebuild that splits the configuration will have to split
the shipped data too, and it may not.

## `Action`

**Contract** — offers an input command to the inventory half first; if it handled it, stop.
Otherwise offer it to the held half. Answers whether anybody handled it.

**Invariants** — first-match-wins, inventory first. An item's inventory behaviour (drop, use)
therefore shadows its held behaviour for any command both claim. No shipped item has such a
collision, but the precedence is the rule.

## `SwitchState` and `OnStateSwitch`

**Contract** — pure delegation to the held half. The inventory half has no state machine.

## `OnEvent`

**Contract** — a network event reaches both halves, inventory first.

## Inventory transitions

**Contract** — four transitions, and the ordering flips between them:

```text
attached to an owner       : held half first, then inventory half
detached from an owner     : inventory half first, then held half
becoming independent       : held half first, then inventory half
finished becoming independent : held half first, then inventory half
moved into the backpack    : inventory half first, then held half
```

**Invariants** — attachment runs the held half first because the held half decides whether the
item is being *taken into a hand* or merely into a container, and the inventory half's
placement logic reads that decision. Detachment runs the inventory half first for the mirror
reason: the item must be out of the grid before the held half tears down a view that is no
longer of anything.

Both independence transitions run the held half first. Becoming a free object in the world
means the first-person view must be dismissed before the object acquires a world position and
a physics body, or the view renders for a frame at the world position.

**Notes** — the "just before destroy" flag on the detaching transition is passed to both
halves. It lets each skip work whose effect nothing will observe, which matters because this
path runs for every item on a creature's corpse.

## `net_Spawn` and `net_Destroy`

**Contract** — spawn both halves, inventory first, and succeed only if **both** succeed.
Destroy runs held half first.

**Invariants** — the spawn is short-circuiting: if the inventory half fails, the held half is
never spawned. A rebuild must not "helpfully" spawn both and then check, because a held half
spawned against a failed inventory half will reference data that was never loaded.

## `ActivateItem` and `DeactivateItem`

**Contract** — pure delegation to the held half. Raising and lowering an item is entirely a
held-item concern.

## `UpdateCL`

**Contract** — both halves update, inventory first.

## Rendering

**Contract** — two entry points with a deliberate crossing:

- The main renderable entry draws the **held** half — the first-person view.
- The secondary entry draws the **inventory** half's renderable — the world model.

**Invariants** — the crossing is the point. An item that is raised is drawn by its held half
through the ordinary render path; the same item's world model is reached through the
secondary path, which is what the renderer calls when the item must appear in the world
(in another player's hands, on the ground, in a corpse's inventory) rather than in this
player's view. Swapping the two draws the world model in front of the camera.

## `use_parent_ai_locations`

**Contract** — the item uses its owner's navigation position **only when it has not been
drawn in the first-person view this frame**.

**Invariants** — this is the one behavioural decision in the file. A raised item's position
for AI purposes — where a sound it makes comes from, where a creature perceives it — must be
the camera, not the owner's navigation vertex, on any frame where the item is visibly at the
camera. The test is a frame-number comparison against the last frame the view transform was
built, which is exactly "was I drawn as a held item this frame".
