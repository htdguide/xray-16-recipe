# src/xrGame/script_particle_action_script.cpp

> Exports the particle channel to the script layer as `particle`.

**Needs** — [`script_particle_action.h`](script_particle_action.h.md) · [`particle_params.h`](particle_params.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Declares the script-visible shape of a particle order. No constant tables: the effect is
named by text, resolved against the authored effect library at play time.

## `script_register`

**Contract** — registers a class named `particle` with six constructors and six methods.

```text
particle()
particle(effect_name, bone_name)
particle(effect_name, bone_name, placement)
particle(effect_name, bone_name, placement, auto_remove)
particle(effect_name, placement)
particle(effect_name, placement, auto_remove)

methods: set_particle(name, auto_remove), set_bone(name),
         set_position(vector), set_angles(vector), set_velocity(vector),
         completed()
```

**Notes**

The six forms are three argument shapes times the binding layer's inability to default an
argument. A rebuild with optional arguments registers two.

Whether a bone name appears is what selects between *follows the entity* and *stands in the
world*, and the choice is made by which constructor a script calls, not by a named
constant. A rebuild would do better to export the goal kind.
