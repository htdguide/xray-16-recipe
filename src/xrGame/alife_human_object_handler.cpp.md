# src/xrGame/alife_human_object_handler.cpp

> The offline inventory manager, stubbed: every method answers "nothing" or does nothing, so an off-screen character never re-equips, never picks anything up and never chooses a weapon.

**Needs** — [`alife_human_object_handler.h`](alife_human_object_handler.h.md) · [`alife_human_object_handler_inline.h`](alife_human_object_handler_inline.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md)
**Used by** — [`alife_human_object_handler.h`](alife_human_object_handler.h.md)
**Tier floor** — T4 as shipped; T3 if implemented.

## Purpose

This type is the offline counterpart of the live re-equipment logic in
[`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md): the same eight equipment categories, the
same choose-and-attach shape, but running against a server record rather than a live
character. Its real implementation is preserved separately in
[`alife_human_object_handler_save.h`](alife_human_object_handler_save.h.md), which is not
compiled.

What ships is a complete set of stubs. The observable consequence is the rule a rebuilder
needs: **an off-screen character's inventory does not change**. It keeps what it had when it
went offline and re-equips only once it is live again.

## The stubbed surface

**Contract** — Every method is a constant:

- `get_available_ammo_count`, in both forms — zero.
- `attach_available_ammo`, `collect_ammo_boxes`, `detach_all`, `update_weapon_ammo`,
  `process_items`, `attach_items`, `choose_group` — no effect.
- `can_take_item`, `choose_fast` — false.
- `choose_equipment`, `choose_weapon`, `choose_food`, `choose_medikit`, `choose_detector`,
  `choose_valuables` — the sentinel meaning "nothing chosen".
- `best_detector`, `best_weapon` — absent.

**Invariants** — The "nothing chosen" sentinel is negative, distinguishing it from a count of
zero items chosen; every caller tests for it. A rebuild returning an optional makes the
distinction explicit.

**Notes** — Callers are written against the real behaviour, so a rebuild implementing this
type gets off-screen re-equipment back without changing any call site. That is worth knowing
before deciding these stubs are dead code: they are a *disabled feature*, wired in, not a
vestige.
