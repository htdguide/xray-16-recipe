# src/xrGame/attachable_item.cpp

> The half of an inventory item that lets it hang visibly on a bone of whoever is carrying it.

**Needs** — [`attachable_item.h`](attachable_item.h.md) · [`attachment_owner.h`](attachment_owner.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Inventory.h`](Inventory.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`attachable_item.h`](attachable_item.h.md); callers name that, not this file.
**Tier floor** — T2: a rigid offset and a visibility flag; the only device contact is handing a visual to the renderer

## Purpose

An artefact clipped to a belt, a detector in a hand, a knife on a thigh: an item that is in
an inventory and *also* on screen, moving with the carrier's animation. This mix-in supplies
that. It carries the authored offset from a named bone, the resolved bone index, and a flag
saying whether the item is currently hanging rather than merely carried.

The counterpart is [`attachment_owner.cpp`](attachment_owner.cpp.md), which owns the list
and does the per-frame placement. The split is along the two questions — "may I hang, and
where on the skeleton" belongs to the item; "what is hanging on me, and where is that bone
this frame" belongs to the carrier.

## State

```text
RECORD CAttachableItem
  item      : reference to the inventory-item half of the same object  # self-cast, never null after construction
  bone_name : text        # the carrier's bone this hangs from, from configuration
  bone_id   : int         # resolved against the CARRIER's skeleton, not the item's
  offset    : matrix      # rigid transform from that bone to the item, from configuration
  enabled   : bool        # is it currently hanging, as opposed to merely carried
```

**Invariants**

- `bone_id` is an index into *the carrier's* skeleton and is meaningless without one. It is
  resolved at attach time and re-resolved whenever the carrier's model changes, because a
  model change renumbers bones and a stale index places the item on the wrong limb or out of
  range.
- An item whose section does not declare an attach position is not attachable at all, and
  its `enabled` is forced false at load. Every other field is then never read.
- `enabled` and "the carrier's list contains me" are one fact stored twice. They are kept in
  step by routing every change through the enable path below; nothing else may set the flag.

## configuration

**Contract** — three keys in the item's section, all or nothing: an angle offset as
heading/pitch/bank, a position offset, and the carrier bone's name. The presence of the
angle-offset key is the test for "is this item attachable"; an item without it loads
normally and simply never hangs.

```text
FUNCTION load_attach_position(section) -> bool
  IF section has no "attach_angle_offset" THEN RETURN false
  offset    = rotation_from_heading_pitch_bank(section.vector("attach_angle_offset"))
  offset.translation = section.vector("attach_position_offset")
  bone_name = section.string("attach_bone_name")
  RETURN true
```

## `reload`

**Contract** — re-reads the attach position for a section. If the item *is* attachable, it
is immediately disabled. Loading is what happens on spawn and on an upgrade, and in both
cases the item must start not-hanging and be enabled deliberately afterwards; an item that
came back from configuration already hanging would attach to a carrier that has not agreed
to it.

## `enable`

**Contract** — the one door in and out of the hanging state, and the only writer of the
flag. With no carrier it records the wish and returns — the item is loose in the world or
in a container, and there is nothing to hang from; the recorded value takes effect when a
carrier appears. With a carrier that can own attachments, it pushes the item into or out of
the carrier's list and flips the item's visibility to match.

```text
FUNCTION enable(value)
  IF no carrier THEN
    enabled = value          # remembered; acted on when a carrier arrives
    RETURN

  owner = carrier AS attachment owner
  IF owner is none THEN RETURN          # carrier cannot hold attachments; the wish is dropped

  IF value AND NOT enabled THEN
    enabled = true
    owner.attach(item)
    item.object.visible = true

  IF NOT value AND enabled THEN
    enabled = false
    owner.detach(item)
    item.object.visible = false
```

**Notes** — when the carrier cannot hold attachments the requested value is *not* recorded,
unlike the no-carrier case. The asymmetry is deliberate in effect if not in intent: a wish
to hang on something that cannot be hung on should not persist and fire later against a
different carrier.

## `can_be_attached`

**Contract** — whether the item's *inventory position* permits hanging, as distinct from
whether it is attachable at all. Three-way:

```text
FUNCTION can_be_attached() -> bool
  IF item is in no inventory THEN RETURN false     # nothing to be attached to
  IF that inventory has no usable belt THEN RETURN true   # belt-less game: hang whatever
  RETURN item.current_place == belt
```

**Notes** — this is where the three games diverge. Two of them give the player a belt with
slots, and only an item actually placed in a belt slot hangs on the model. The third has no
belt in its interface, and there the rule degrades to "anything carried may hang". A
rebuild supporting all three data sets needs both branches; the belt's existence is a
property of the inventory, read from configuration, not a build-time choice.

## `OnH_A_Chield` — becoming carried

**Contract** — called when the object is given a parent. If that parent is an inventory
owner that already lists this item as attached, the item is made visible. This handles the
order in which a save is loaded and a network spawn arrives: the carrier can learn it owns
an attached item before the item itself exists, and this is where the item catches up.

## `OnH_A_Independent` — becoming loose

**Contract** — called when the object loses its parent. Disables the attachment
unconditionally: an item lying on the ground hangs from nothing.

## `afterAttach` / `afterDetach`

**Contract** — the carrier calls these once the list has actually changed. They register and
unregister the item with the per-frame scheduler. An item hanging on a moving body needs its
own update — its sound, its particle effect, its condition decay all keep running while it
is on the belt — and an item that is not hanging must not consume an update slot.

**Invariants** — exactly balanced: every attach eventually pairs with a detach. An unbalanced
pair leaves a destroyed item registered with the scheduler.

## `renderable_Render`

**Contract** — submits the item's visual at the item's own world transform, which the
carrier's placement pass has already written. The item renders as part of the carrier's
render call rather than on its own, so that it is culled, lit and sorted with the body it
hangs on instead of being independently visible through walls that hide the carrier.

## `use_parent_ai_locations`

**Contract** — false while hanging, true otherwise. An item in a pocket may borrow its
carrier's navigation vertex and level position, because it is at the carrier's position. An
item hanging on a bone is metres from the carrier's origin and up in the air, so it must
resolve its own — otherwise every belt artefact reports the carrier's footprint as its
location, and anything that reasons about where an artefact is gets the wrong answer.

## `_construct`

**Contract** — links the two halves of the same object: the attachable mix-in finds the
inventory-item mix-in on the same instance and caches the reference, then hands back the
concrete object. Called once, before anything else.

**Notes** — this is the cost of assembling an item out of independent mix-ins with no common
base: each mix-in has to find its siblings on the same object at construction. A rebuild
that composes an item as one type with an optional attachment component does not need any of
it.

## live offset tuning

**Notes** — the class carries a global "currently tuned item" reference and a set of nudge
operations that rotate and translate that item's offset by a delta on one axis. These are
wired to console commands in debug builds, and they exist because placing a knife on a thigh
convincingly is an eyeballing job: an artist runs the game, selects the item, nudges it
until it looks right and copies the resulting numbers back into configuration. A rebuild
that omits the tool must still expect the authored offsets to have been produced this way —
they are not derived from anything and cannot be recomputed.
