# src/xrServerEntities/xrServer_Objects_ALife_script2.cpp

> Exports the world-furniture records: projector, helicopter, car, breakable, climbable, mounted weapon, team base.

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The second part of the alife export, split for build time. Seven records, each registered
with its name and its bases and **no additional surface at all** — a script may subclass any
of them and override its serialization, and that is the whole point of their being here.

- `cse_alife_object_projector` — a spotlight.
- `cse_alife_helicopter` — a dynamic visual object that is also a motion path and a ragdoll.
  The only record in the chapter combining all three.
- `cse_alife_car` — a driveable vehicle: a visual object with a ragdoll.
- `cse_alife_object_breakable` — something that shatters.
- `cse_alife_object_climable` — a ladder. Exported at the **abstract** level, not the
  dynamic-alife one: a ladder is a shape and a placement, never simulated.
- `cse_alife_mounted_weapon` — a fixed gun a creature can use.
- `cse_alife_team_base_zone` — a restrictor marking a multiplayer team's base.

## Notes

**These records are exported so mods can replace them, not so mods can read them.** Nothing
about a helicopter's flight path or a car's state reaches script through this file; a mod
that wants to change how a car serializes declares a script subclass and overrides the two
state methods. That is the whole exported contract, and it is why the file is a list rather
than a set of surfaces.
