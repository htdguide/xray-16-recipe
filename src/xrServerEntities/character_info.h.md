# src/xrServerEntities/character_info.h

> Declares the per-person profile: which authored character a creature is, and the current rank, reputation, community and sympathy that follow from it.

**Needs** — [`character_info_defs.h`](character_info_defs.h.md) · [`shared_data.h`](shared_data.h.md) · [`xml_str_id_loader.h`](xml_str_id_loader.h.md) · [`specific_character.h`](specific_character.h.md) · [`xrGame/character_community.h`](../xrGame/character_community.h.md) · [`xrGame/character_rank.h`](../xrGame/character_rank.h.md) · [`xrGame/character_reputation.h`](../xrGame/character_reputation.h.md)
**Used by** — [`AI_PhraseDialogManager.cpp`](../xrGame/AI_PhraseDialogManager.cpp.md) · [`HUDTarget.cpp`](../xrGame/HUDTarget.cpp.md) · [`InventoryOwner.cpp`](../xrGame/InventoryOwner.cpp.md) · [`InventoryOwner.h`](../xrGame/InventoryOwner.h.md) · [`Level_bullet_manager_firetrace.cpp`](../xrGame/Level_bullet_manager_firetrace.cpp.md) · [`actor_communication.cpp`](../xrGame/actor_communication.cpp.md) · [`script_game_object_inventory_owner.cpp`](../xrGame/script_game_object_inventory_owner.cpp.md) · [`trade2.cpp`](../xrGame/trade2.cpp.md) · [`UITalkWnd.cpp`](../xrGame/ui/UITalkWnd.cpp.md) · [`xrgame_dll_detach.cpp`](../xrGame/xrgame_dll_detach.cpp.md) · [`character_info.cpp`](character_info.cpp.md) · [`xrServer_Objects_ALife.cpp`](xrServer_Objects_ALife.cpp.md) · [`xrServer_Objects_ALife_Monsters.cpp`](xrServer_Objects_ALife_Monsters.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`character_info.cpp`](character_info.cpp.md).

Two things are declared here and are worth naming before reading that file. A **profile** is
a *template*: a requirement for a character (this class, around this rank, around this
reputation) or a direct reference to one specific character. A **specific character** is the
authored individual — a name, a portrait, a biography, a dialogue set, a starting faction.
A creature record stores the profile identifier and resolves it to a specific character when
it is first created, which is how "spawn a bandit of about this rank" becomes a named person
with a face.

## Exported units

- **character profile** — the shared, XML-loaded template: a specific-character reference or
  a (class, rank, reputation) requirement.
- **character info** — the per-entity instance: the profile identifier it was created from,
  the specific character it resolved to, its current rank, reputation, community and
  sympathy, and its opening dialogue.
- `load_profile(identifier)` — bring in the profile template.
- `bind_to(record)` — fill this from a trader record's stored fields.
- `resolve_specific_character(identifier)` — attach an authored individual, filling in
  anything the record left unset.
- `save` / `load` — the one field of this that reaches the entity record.
- the read-only surface scripts and the UI use: name, biography, portrait, community, rank,
  reputation, sympathy, opening dialogue, the actor-dialogue list.
