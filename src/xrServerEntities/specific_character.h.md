# src/xrServerEntities/specific_character.h

> The authored individual: everything about a named character that is written by hand rather than simulated.

**Needs** — [`specific_character.cpp`](specific_character.cpp.md) · [`character_info_defs.h`](character_info_defs.h.md) · [`shared_data.h`](shared_data.h.md) · [`xml_str_id_loader.h`](xml_str_id_loader.h.md)
**Used by** — [`PDA.cpp`](../xrGame/PDA.cpp.md) · [`alife_trader_abstract.cpp`](../xrGame/alife_trader_abstract.cpp.md) · [`xrgame_dll_detach.cpp`](../xrGame/xrgame_dll_detach.cpp.md) · [`character_info.cpp`](character_info.cpp.md) · [`character_info.h`](character_info.h.md) · [`specific_character.cpp`](specific_character.cpp.md) · [`xrServer_Objects_ALife_Monsters.cpp`](xrServer_Objects_ALife_Monsters.cpp.md) · [`xrServer_Objects_ALife_Monsters_script.cpp`](xrServer_Objects_ALife_Monsters_script.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the record of an authored character profile — the thing a record names when it says
"this stalker is *Sidorovich*" — and the accessors over it. The loading is in
[`specific_character.cpp`](specific_character.cpp.md); the fields are listed there with what
each one is for.

Two mechanisms are mixed in and both are load-bearing to how a profile behaves. The profile
is **shared** ([`shared_data.h`](shared_data.h.md)): every entity naming the same profile
points at one parsed copy, loaded on first demand. And it is **indexed**
([`xml_str_id_loader.h`](xml_str_id_loader.h.md)): profiles live in a set of authored XML
files, are named by string, and are additionally numbered so a record can hold a small
integer instead of a name.

The sharing is configured **not to free itself** when the last user goes away, so a profile
parsed once stays parsed for the process's lifetime. Profiles are re-resolved at every level
change and there are a few thousand of them; re-parsing would be visible.

## the exported units

- `Load(id)` — bind this handle to the profile with that identifier, parsing it if this is
  the first demand.
- The read surface, one accessor per field: display name, biography text, faction, rank,
  reputation, visual model, starting inventory, behaviour section, voice-set prefix, panic
  threshold, hit-probability factor, crouch style, "is a repair mechanic", critical-wound
  weights, inventory icon, preferred terrain section, and the money range.

The record itself — what a profile *is* — is in
[`specific_character.cpp`](specific_character.cpp.md).
