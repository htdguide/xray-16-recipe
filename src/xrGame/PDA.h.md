# src/xrGame/PDA.h

> Declares the personal data assistant implemented in [`PDA.cpp`](PDA.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [`PdaMsg.h`](PdaMsg.h.md) · [`InfoPortionDefs.h`](../xrServerEntities/InfoPortionDefs.h.md) · [`character_info_defs.h`](../xrServerEntities/character_info_defs.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`PDA.cpp`](PDA.cpp.md)
**Used by** — [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`InfoDocument.cpp`](InfoDocument.cpp.md) · [`InventoryOwner.cpp`](InventoryOwner.cpp.md) · [`PDA.cpp`](PDA.cpp.md) · [`UIZoneMap.cpp`](UIZoneMap.cpp.md) · [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) · [`torch_script.cpp`](torch_script.cpp.md) · [`UIFactionWarWnd.cpp`](ui/UIFactionWarWnd.cpp.md) · [`UIHudStatesWnd.cpp`](ui/UIHudStatesWnd.cpp.md) · [`UIPdaWnd.cpp`](ui/UIPdaWnd.cpp.md) · [`UITalkWnd.cpp`](ui/UITalkWnd.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CPda`. The declaration's own content is that the class carries **two roles**: it is an
inventory item, and it implements the engine's *touch* sense. The second is what makes a device
notice nearby characters at all, and a rebuild must provide that attachment point — perception
in this engine is an opt-in interface an object implements, not a service it queries.

Substance is in [`PDA.cpp`](PDA.cpp.md).

Exported units:

- `CPda` — the device.
- `net_Spawn` / `Load` / `net_Destroy` / `OnH_A_Chield` / `OnH_B_Independent` /
  `shedule_Update` — the lifecycle and the low-rate sensing update.
- `feel_touch_new` / `feel_touch_delete` / `feel_touch_contact` — the three touch-sense hooks:
  what to track, and what to do when something enters or leaves.
- `TurnOn` / `TurnOff` / `IsOn` / `IsOff` / `IsActive` — the on/off state, which is decided by
  who is holding the device rather than by the user.
- `GetOriginalOwnerID` / `GetOriginalOwner` / `GetOwnerObject` — the identity of the character
  the device was issued to, which is what gates it being on.
- `ActivePDAContacts` / `ActiveContactsNum` / `GetPdaFromOwner` / `UpdateActiveContacts` — the
  contact list.
- `PlayScriptFunction` / `CanPlayScriptFunction` — the recorded-message playback hook.
- `save` / `load` — persist the composed display name.
- `PDA_LIST` — the list alias used wherever several devices are handled together.
