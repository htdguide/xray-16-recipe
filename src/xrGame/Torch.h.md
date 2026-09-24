# src/xrGame/Torch.h

> Declares the flashlight and its night-vision companion, implemented in [`Torch.cpp`](Torch.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [`HudSound.h`](HudSound.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`ActorHelmet.cpp`](ActorHelmet.cpp.md) · [`ActorInput.cpp`](ActorInput.cpp.md) · [`CustomOutfit.cpp`](CustomOutfit.cpp.md) · [`Torch.cpp`](Torch.cpp.md) · [`Weapon.cpp`](Weapon.cpp.md) · [`object_handler.cpp`](object_handler.cpp.md) · [`torch_script.cpp`](torch_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CTorch`, the flashlight inventory item that owns a beam, a fill light and a lens
glow, and `CNightVisionEffector`, the small object that drives the night-vision
post-process effect and its four sounds. Substance is in [`Torch.cpp`](Torch.cpp.md).

It also fixes the torch's **replicated flag set** — three bits in one byte: the light is
on, night vision is on, the item is attached. That byte is the whole of the torch's network
state and is frozen against the protocol.

Exported units:

- `CTorch` — the item.
- `Load`, `net_Spawn`, `net_Destroy`, `net_Export`, `net_Import`, `UpdateCL` — the
  lifecycle; the light's own description is read from the model's embedded user data, not
  from the item's section.
- `Switch` (toggle and explicit), `torch_active` — the light switch.
- `SwitchNightVision` (toggle and explicit), `GetNightVisionStatus`, `GetNightVision` —
  the night-vision switch.
- `OnH_A_Chield`, `OnH_B_Independent`, `afterDetach`, `enable`, `can_be_attached` — the
  attachment lifecycle, every exit from which puts the light out.
- `create_physic_shell`, `activate_physic_shell`, `setup_physic_shell` — a dropped torch
  gets a real body, built by the shell-holder base directly.
- `use_parent_ai_locations` — true only while unheld: a carried torch has no navigation
  position of its own.
- `script_register` — declares the type to Lua.
- `CNightVisionEffector` — the night-vision effect driver: `Start`, `Stop`, `IsActive`,
  `OnDisabled`, `PlaySounds`, over the four cues (start, stop, idle hum, malfunction).
