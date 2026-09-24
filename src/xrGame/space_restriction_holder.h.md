# src/xrGame/space_restriction_holder.h

> Declares the level's registry of restrictor volumes and the two level-wide default restriction lists.

**Needs** — [`space_restriction_holder_inline.h`](space_restriction_holder_inline.h.md) · [`xrServerEntities/restriction_space.h`](../xrServerEntities/restriction_space.h.md)
**Used by** — [`space_restriction.h`](space_restriction.h.md) · [`space_restriction_composition.cpp`](space_restriction_composition.cpp.md) · [`space_restriction_composition.h`](space_restriction_composition.h.md) · [`space_restriction_holder.cpp`](space_restriction_holder.cpp.md) · [`space_restriction_holder_inline.h`](space_restriction_holder_inline.h.md) · [`space_restriction_manager.cpp`](space_restriction_manager.cpp.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md)
**Tier floor** — T2: a declaration over a keyed registry

## Purpose

Declares the surface implemented in [`space_restriction_holder.cpp`](space_restriction_holder.cpp.md)
and [`space_restriction_holder_inline.h`](space_restriction_holder_inline.h.md). It also
names the handle type the whole family passes around — a reference-counted pointer to a
bridge — which is why nearly every file in the family includes this one.

## Exported units

- `restriction(names)` — the shared handle for a normalized name list.
- `register_restrictor`, `unregister_restrictor` — spawn and despawn of restrictor geometry.
- `default_out_restrictions`, `default_in_restrictions` — the level-wide lists.
- `on_default_restrictions_changed` — demanded of a subclass; fires when either default list changes.
- `normalize_string` — the canonicalization that makes a name list a key.
- `collect_garbage`, `clear` — reclamation and teardown.

## Notes

The reclamation delay (five minutes of world clock) and the per-list name cap (128) are
declared here; both are explained in the implementation twin.

`on_default_restrictions_changed` is the only thing this type demands of its subclass, and
it is what makes the registry usable without knowing that entities exist: the registry
knows restrictors, the manager knows entities, and this one notification is the whole seam
between them.
