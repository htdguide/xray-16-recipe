# src/xrGame/material_manager_inline.h

> The two material indices and the pairing they select.

**Needs** — [`material_manager.h`](material_manager.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md)
**Used by** — [`material_manager.h`](material_manager.h.md)
**Tier floor** — T3: a table lookup

## Purpose

Carries three accessors out of the declaration. Two are field reads; the third is not, and it
is the reason this file is worth a page.

## State

`Stateless.`

## `last_material_idx` · `self_material_idx`

**Contract** — the material the creature is currently standing on, and the material the
creature itself is made of. The first is written by the physics movement control, not by this
manager.

## `get_current_pair`

**Contract** — returns the material-pair description for this creature on this ground: the
sounds, particles and marks that the combination of the two materials produces. **Forces the
movement control to refresh the material underfoot first**, so the answer reflects where the
creature is *now* rather than where it was at the last physics step.

**Invariant** — the forced refresh is the whole point of the call. A caller reading the two
indices separately and looking the pair up itself gets last step's ground, which is visibly
wrong at the moment a creature steps from one surface onto another — exactly when the pair is
asked for.

**Notes** — the lookup is by *ordered* pair, the creature's own material first. Material
pairings are directional in the data: a boot on metal is a different sound from metal on a
boot.
