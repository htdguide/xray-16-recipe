# src/xrServerEntities/alife_human_brain.h

> Declares the offline brain a human creature carries: the monster brain plus the preferences and the money that make a stalker trade and equip itself while nobody is looking.

**Needs** — [`alife_monster_brain.h`](alife_monster_brain.h.md) · [`alife_human_brain_inline.h`](alife_human_brain_inline.h.md)
**Used by** — [`alife_human_abstract.cpp`](../xrGame/alife_human_abstract.cpp.md) · [`alife_human_brain_script.cpp`](../xrGame/alife_human_brain_script.cpp.md) · [`ef_primary.cpp`](../xrGame/ef_primary.cpp.md) · [`stalker_alife_task_actions.cpp`](../xrGame/stalker_alife_task_actions.cpp.md) · [`stalker_property_evaluators.cpp`](../xrGame/stalker_property_evaluators.cpp.md) · [`alife_human_brain.cpp`](alife_human_brain.cpp.md) · [`alife_human_brain_inline.h`](alife_human_brain_inline.h.md) · [`xrServer_Objects_ALife_Monsters.cpp`](xrServer_Objects_ALife_Monsters.cpp.md) · [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`xrServer_Objects_ALife_Monsters_script4.cpp`](xrServer_Objects_ALife_Monsters_script4.cpp.md)
**Tier floor** — T1: two of its fields are serialized into the entity record at fixed widths.

## Purpose

Declares the surface implemented in
[`alife_human_brain.cpp`](alife_human_brain.cpp.md).

## Exported units

- **the human brain** — extends the monster brain with an *object handler* (the offline
  inventory manipulator: what this stalker picks up, trades and wields) and with the record
  state below.
- `equipment_preferences` — a fixed-length array of per-equipment-kind taste values.
- `weapon_preferences` — a fixed-length array of per-weapon-kind taste values.
- `money` — the amount this human is carrying.
- `on_state_write` / `on_state_read` — the brain's slice of the entity record, and the part
  of this chapter's serialization contract that lives outside the record classes.
- `objects()` — the offline inventory handler.
