# src/xrGame/HolderEntityObject.h

> Declares the generic mountable prop, implemented in [`HolderEntityObject.cpp`](HolderEntityObject.cpp.md).

**Needs** — [`holder_custom.h`](holder_custom.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md)
**Used by** — [`HolderEntityObject.cpp`](HolderEntityObject.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CHolderEntityObject` as a physics-shell holder that also implements the engine's
holder interface — the contract for "something the player can get into". Substance is in
[`HolderEntityObject.cpp`](HolderEntityObject.cpp.md).

Read as a *list*, this header is the holder interface's full surface, which is its main
value: a rebuild implementing any holder must answer everything below.

Exported units:

- `CHolderEntityObject` — the prop.
- `Load`, `net_Spawn`, `net_Destroy`, `net_Export`, `net_Import`, `UpdateCL`,
  `renderable_Render` — the ordinary entity lifecycle.
- `Hit` — damage, refused while occupied.
- `Use` — whether this holder may be entered, given where the player is standing and
  looking.
- `attach_Actor`, `detach_Actor`, `attach_actor_script`, `detach_actor_script` — mount
  and dismount, with the script forms able to bypass the enter and exit locks.
- `Camera`, `cam_Update` — the camera the occupant looks through, and its per-frame
  placement.
- `allowWeapon`, `HUDView`, `ExitPosition`, `GetInventory` — what a holder tells the
  actor about itself: whether a weapon may be held, whether the view is first-person,
  where the occupant is put down, and whether it carries an inventory of its own.
- `OnMouseMove`, `OnKeyboardPress`, `OnKeyboardRelease`, `OnKeyboardHold`, and the four
  controller hooks — every input event a holder must accept and may discard.
- `Action`, `SetParam` — the command and analogue-parameter channels.
- `cast_holder_custom` — the capability answer that makes this object a holder.
- `BoneCallbackX`, `BoneCallbackY` — declared, empty, unreferenced; vestigial from the
  vehicle this was derived from.
