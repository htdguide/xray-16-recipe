# src/xrGame/alife_switch_manager.h

> Declares the online/offline transition layer, implemented in [`alife_switch_manager.cpp`](alife_switch_manager.cpp.md) and [`alife_switch_manager_inline.h`](alife_switch_manager_inline.h.md).

**Needs** — [`alife_simulator_base.h`](alife_simulator_base.h.md) · [`alife_switch_manager_inline.h`](alife_switch_manager_inline.h.md)
**Used by** — [`alife_switch_manager.cpp`](alife_switch_manager.cpp.md) · [`alife_switch_manager_inline.h`](alife_switch_manager_inline.h.md) · [`alife_update_manager.cpp`](alife_update_manager.cpp.md) · [`alife_update_manager.h`](alife_update_manager.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeSwitchManager`, the layer that owns the promotion and demotion of entities
and the two radii that decide when. Substance is in
[`alife_switch_manager.cpp`](alife_switch_manager.cpp.md); the radius derivation is in
[`alife_switch_manager_inline.h`](alife_switch_manager_inline.h.md).

Two remarks a rebuild should act on.

The layer **inherits a random stream** rather than holding one, which the source itself
flags as wrong. It is wrong for a reason worth naming: inheriting makes every method of
the layer look like a randomness consumer, and hides that only a small part of the
switching logic draws at all. A rebuild should hold the stream as a field.

The scratch child list is a member because it is used across one demotion; making it local
is safe and clearer.

Exported units:

- `CALifeSwitchManager` — construct from a server and a configuration section, which
  supplies the nominal radius and the hysteresis factor.
- `switch_object` — the per-entity entry point: reap, reconcile, decide, reap again.
- `try_switch_online` / `try_switch_offline` — the two decisions.
- `switch_online` / `switch_offline` — the two transitions, delegating to the entity.
- `add_online` / `remove_online` — materialize and dematerialize the live client object.
- `synchronize_location` — reconcile a record with reality before deciding.
- `online_distance`, `offline_distance`, `switch_distance` — read the three radii.
- `set_switch_distance`, `set_switch_factor` — change them; both re-derive the pair.
