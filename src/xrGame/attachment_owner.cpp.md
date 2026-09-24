# src/xrGame/attachment_owner.cpp

> The half of a carrier that holds a list of items hanging on its skeleton and re-places them every time its bones are evaluated.

**Needs** — [`attachment_owner.h`](attachment_owner.h.md) · [`attachable_item.h`](attachable_item.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — [`attachment_owner.h`](attachment_owner.h.md)
**Tier floor** — T2: a small list and a transform composition per attached item, run inside pose evaluation

## Purpose

The carrier's side of the attachment system described in
[`attachable_item.cpp`](attachable_item.cpp.md). It answers three questions: which item
sections this carrier will accept, which items are currently hanging on it, and — once per
frame, after the skeleton is posed — where each of them is in the world.

Its one non-obvious job is the placement pass. An attached item is a full world object with
its own transform, not a child node of the skeleton, because it has to collide, be picked
up, be hit and be saved like any other object. So its transform has to be *pushed* to it
after the carrier's bones are final, and that is what the visual callback below does.

## State

```text
RECORD CAttachmentOwner
  accepted_sections : list<text>                    # from configuration; what may hang here
  attached          : list<reference to attachable item>   # non-owning; the items own themselves
```

**Invariants**

- At most one attached item per *section*. Two detectors on one belt would occupy the same
  bone with the same offset and render inside each other, so the acceptance test rejects the
  second.
- The list is non-owning. Every entry must be removed before the carrier is destroyed; the
  carrier asserts the list is empty at network-destroy time rather than clearing it, because
  a carrier dying with items still attached means those items were leaked or are about to
  reference a dead skeleton. Turning that into a silent cleanup would hide the real bug,
  which is an item whose detach never ran.
- The placement callback is registered with the carrier's model exactly while the list is
  non-empty — added when it becomes non-empty, removed when it becomes empty. A carrier with
  nothing attached pays nothing per frame.
- `bone_id` on every entry must match the carrier's *current* model. Changing the carrier's
  model invalidates all of them at once.

## configuration

**Contract** — one key in the carrier's section, a comma-separated list of item section
names this carrier accepts. Absent means the carrier accepts nothing, and the list is
cleared rather than left as it was — a carrier reloaded onto a section with no attachments
must not keep the old section's.

## the placement pass

**Contract** — runs as a callback on the carrier's model, after its bones are evaluated and
before it is rendered. For each attached item, composes the item's world transform from
three parts in this order: the carrier's world transform, then the bone's transform within
the model, then the item's authored offset.

```text
FUNCTION place_attachments(carrier)
  FOR EACH a IN carrier.attached
    a.object.transform = carrier.transform
                         COMPOSED WITH carrier.bone_transform(a.bone_id)
                         COMPOSED WITH a.offset
```

**Invariants** — this must run *after* the skeleton is posed. Running it before leaves every
attached item one frame behind the body it hangs on, which on a running character is a
visible lag of the belt against the hips. Hooking the model's own post-pose notification
rather than the game update is what guarantees the order.

**Notes** — the callback is a free function that recovers the carrier from a parameter the
model carries. That indirection is a consequence of the model interface owning the hook; a
rebuild whose pose evaluation can call a method on the object needs none of it.

**Notes** — nothing here is per-item state. A rebuild that makes an attachment a child
transform of the bone gets the same result declaratively, at the cost of the item no longer
being an independent world object — which it must remain, so the push is the right shape.

## `attach`

**Contract** — adds an item to the list if it is acceptable and not already there. Resolves
the item's bone name against the carrier's *current* skeleton, registers the placement
callback if this is the first attachment, makes the item visible, and tells the item its
attach has completed.

```text
FUNCTION attach(item)
  IF any entry has the same entity identifier THEN RETURN     # already attached
  IF NOT can_attach(item) THEN RETURN

  IF attached is empty THEN register the placement callback with the model
  item.bone_id = carrier.model.bone_index(item.bone_name)
  append item to attached
  item.object.visible = true
  item.after_attach()
```

**Notes** — the duplicate check is by entity identifier and returns silently. The source
says outright that this is papering over a double-attach elsewhere rather than fixing it. A
rebuild should make double attachment impossible rather than tolerated — but should expect
the shipped data and scripts to try it.

**Notes** — the bone name failing to resolve is not checked. A configuration file naming a
bone the carrier's model does not have produces an out-of-range index that the placement
pass then reads. This is a real gap; a rebuild should reject the attach.

## `detach`

**Contract** — removes the item with the matching entity identifier, tells it its detach has
completed, and if that emptied the list, unregisters the placement callback and hides the
item. Silently does nothing if the item is not attached.

**Notes** — the visibility is only cleared when the list becomes *empty*, not when the item
is removed. Detaching one of two attached items therefore leaves it visible, still drawn at
its last placed transform until something else hides it. This is a bug, and it is quiet
because carriers in the shipped games rarely hold two attachments at once. A rebuild should
hide the detached item, and unregister the callback independently.

**Notes** — the after-detach notification fires *after* the entry is removed, while attach
fires after insertion. Both mean "the list is now correct"; the item's scheduler
registration depends on it, so the ordering is load-bearing in both directions.

## `can_attach`

**Contract** — three conditions, all required: the item is attachable and currently enabled
and its inventory position permits hanging; the item's section is in this carrier's accepted
list; and no item of that section is already attached.

```text
FUNCTION can_attach(item) -> bool
  a = item AS attachable item
  IF a is none OR NOT a.enabled OR NOT a.can_be_attached() THEN RETURN false
  IF item.section NOT IN accepted_sections THEN RETURN false
  IF attached_item(item.section) exists THEN RETURN false
  RETURN true
```

## `reattach_items`

**Contract** — re-resolves every attached item's bone index against the carrier's current
skeleton. Called after the carrier's model is replaced — a character changing outfit, a
corpse switching to a different visual — because bone indices are per-model and every cached
one is stale the moment the model changes. Nothing is attached or detached.

## `attachedItem`

**Contract** — three lookups over the list, by entity identifier, by class identifier and by
section name. Linear scans; the list is never more than a handful of entries, so the absence
of an index is a correct decision rather than an oversight.

**Notes** — the lookup by section additionally skips items marked invalid, while the other
two do not. An item can be marked invalid while still listed — it has been consumed or
destroyed but its removal has not yet run — and the section lookup is the one used to decide
whether a *new* item of that section may attach, where treating a dying item as present
would block its replacement.

## `reinit` and `net_Destroy`

**Contract** — both assert the list is empty. `reinit` runs when an entity is recycled for a
new spawn and must not inherit the previous occupant's attachments; `net_Destroy` runs at
the end of the entity's life. Neither clears the list, for the reason given under State.

## `renderable_Render`

**Contract** — forwards the render call to every attached item, so their visuals are
submitted as part of the carrier's. See [`attachable_item.cpp`](attachable_item.cpp.md) for
why they are not submitted independently.
