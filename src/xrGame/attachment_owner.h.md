# src/xrGame/attachment_owner.h

> Declares the attachment-carrier mix-in implemented in [`attachment_owner.cpp`](attachment_owner.cpp.md).

**Needs** — [`attachment_owner.cpp`](attachment_owner.cpp.md) · [`attachable_item.h`](attachable_item.h.md)
**Used by** — [`InventoryOwner.h`](InventoryOwner.h.md) · [`attachable_item.cpp`](attachable_item.cpp.md) · [`attachment_owner.cpp`](attachment_owner.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`attachment_owner.cpp`](attachment_owner.cpp.md). A carrier class — the actor, a
non-player character — inherits this alongside its other mix-ins.

Exported units:

- **`cast_game_object`** — required of every implementor: the carrier must be able to hand
  back its game-object identity, since the placement pass and the model live there.
- **`cast_attachment_owner`** — the downcast hook by which any object is asked "can you hold
  attachments".
- **`reload`** — read the accepted section list from configuration.
- **`attach`, `detach`, `can_attach`** — list maintenance and the acceptance rule.
- **`attached`** — by item and by section name.
- **`attachedItem`** — lookup by entity identifier, class identifier or section.
- **`attached_objects`** — read access to the list.
- **`reattach_items`** — re-resolve bone indices after the carrier's model changes.
- **`renderable_Render`** — submit every attachment's visual with the carrier's.
- **`reinit`, `net_Destroy`** — lifecycle points that require the list to be empty.
