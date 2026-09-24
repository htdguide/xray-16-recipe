# src/xrGame/stalker_animation_offsets.hpp

> Declares the per-animation aim-offset table implemented in [`stalker_animation_offsets.cpp`](stalker_animation_offsets.cpp.md).

**Needs** — [`stalker_animation_offsets.cpp`](stalker_animation_offsets.cpp.md) · [`xrServerEntities/xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`stalker_animation_offsets.cpp`](stalker_animation_offsets.cpp.md)
**Tier floor** — T3: a lookup table keyed by name

## Purpose

Declares `animation_offsets`, a small read-mostly map from animation identifier to a
yaw/pitch pair. It exists so that the direction a stalker's weapon actually points during
a given animation can be corrected per animation from configuration, instead of being
baked into the model.

Exported units:

- `animation_offsets` — the table. Final: nothing derives from it.
- `load(section)` — fills the table from one configuration section.
- `offsets(animation_id)` — the yaw/pitch correction for one animation, or a zero
  rotation when the animation is not listed.

**Notes** — the map is ordered by the *identity* of the interned name rather than by its
text. That makes lookup a pointer comparison instead of a string comparison, and it makes
the iteration order of the table depend on where strings happened to land in the intern
pool — which is fine only because nothing iterates it. A rebuild is free to key by the
string itself; the price is a slower per-frame lookup, and the benefit is a table that
iterates deterministically.
