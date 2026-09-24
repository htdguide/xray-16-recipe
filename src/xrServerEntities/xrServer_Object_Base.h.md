# src/xrServerEntities/xrServer_Object_Base.h

> Declares the base entity record and the four-way serialization surface implemented in [`xrServer_Object_Base.cpp`](xrServer_Object_Base.cpp.md).

**Needs** — [`xrServer_Objects_Abstract.h`](xrServer_Objects_Abstract.h.md) · [`script_value_container.h`](script_value_container.h.md) · [`alife_space.h`](alife_space.h.md) · [`Common/object_interfaces.h`](../Common/object_interfaces.h.md)
**Used by** — [`base_client_classes_wrappers.h`](../xrGame/base_client_classes_wrappers.h.md) · [`console_commands_mp.cpp`](../xrGame/console_commands_mp.cpp.md) · [`game_sv_capture_the_artefact.h`](../xrGame/game_sv_capture_the_artefact.h.md) · [`game_sv_item_respawner.h`](../xrGame/game_sv_item_respawner.h.md) · [`game_sv_mp.cpp`](../xrGame/game_sv_mp.cpp.md) · [`script_properties_list_helper.cpp`](script_properties_list_helper.cpp.md) · [`xrServer_Objects.h`](xrServer_Objects.h.md)
**Tier floor** — T1: the field list is a byte layout.

## Purpose

Declares the surface whose contracts live in
[`xrServer_Object_Base.cpp`](xrServer_Object_Base.cpp.md). It also fixes the *composition*
of the base record, which is the part worth naming here: an entity record is simultaneously
a **server entity** (the editor- and level-facing interface), a **serializable object** (the
four serialization directions) and a **script value container** (a bag of script-declared
fields that serialize with it). Those three roles are orthogonal and a rebuild should keep
them orthogonal.

## Exported units

- `CPureServerObject` — the empty four-direction serialization base: file read/write, packet
  read/write.
- `CSE_Abstract` — the base entity record. Carries the prefix fields, the class identifier,
  the placement, the spawn-flag set, the child list, the client-side opaque blob and the
  custom-data overlay; declares the spawn read/write pair, the editor property hooks, and the
  cast family.
- `ESpawnFlags` — the removed repeat-spawn control bits, retained because the record still
  reserves them.
- `script_server_object_version` — the data-supplied script version stamped into every
  record.
