# src/xrGame/map_script.cpp

> Exports the map manager and the map marker to the script virtual machine.

**Needs** — [`map_manager.h`](map_manager.h.md) · [`map_location.h`](map_location.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Quest scripts place, move, label and remove map markers. This file is the surface they do it
through — deliberately narrower than the native one.

## State

`Stateless.`

## `CMapManager::script_register`

**Contract** — registers `CMapManager` with four operations: remove every marker belonging to
an entity, remove one marker by handle, and disable every pointer. Creation is **not**
exported here; scripts create markers through the `level` namespace's map functions in
[`level_script.cpp`](level_script.cpp.md), which take names rather than handles.

**Notes** — a default constructor is exported, which is meaningless for a manager that the
level owns. Constructing one from script produces an object with no registry behind it; the
export is an accident of the registration idiom, not a supported operation.

## `CMapLocation::script_register`

**Contract** — registers `CMapLocation` with its presentation surface: the hint (read, write
and whether it is enabled), the pointer switches, the spot switches, the highlight, the spot
size, the collidable and user-defined flags, the level name, the map position, the last world
position, and the entity identifier.

No constructor is exported: a marker only ever exists as something the manager made, and a
script-constructed one would not be in the registry.

**Invariant** — the *serializable* flag is not exported, even though it is what decides
whether a marker survives a save. Scripts choose it at creation time instead, by calling the
serializing variant of the add function. A rebuild must keep that asymmetry or scripts will
start marking runtime markers as persistent.

**Notes** — the hint setter and the level-name getter are wrapped rather than bound directly,
because the native forms take and return the engine's interned string type and scripts deal
in plain text. That is a binding detail; what it records is that the script surface speaks
text, not handles to interned strings.
