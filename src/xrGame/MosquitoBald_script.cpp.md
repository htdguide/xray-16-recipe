# src/xrGame/MosquitoBald_script.cpp

> Exports three anomaly classes to the script virtual machine.

**Needs** — [`MosquitoBald.h`](MosquitoBald.h.md) · [`TorridZone.h`](TorridZone.h.md) · [`ZoneCampfire.h`](ZoneCampfire.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

One registration entry point declares three anomaly classes to Lua. Two are bare types,
registered by their engine name as subclasses of the game object facade with a no-argument
constructor: scripts get the type for identification and casting and nothing more. The third,
the campfire, is the only zone in the game a script is expected to operate, so it exports three
methods.

The grouping of three classes into one registration function is an accident of where someone
put the code; a rebuild should register each class where it is defined.

## State

`Stateless.`

## `CMosquitoBald::script_register`

**Contract** — registers into the script virtual machine, under the names `CTorridZone`,
`CMosquitoBald` and `CZoneCampfire`, three types each deriving from the game object facade and
each default-constructible from script. Runs once at script-engine bring-up. Names are frozen
by conformance criterion 10.

The campfire additionally exports:

- `turn_on` / `turn_off` — light and extinguish it. These are the script-facing wrappers, not
  the engine-internal ones, because the script call must also fire the state change the rest of
  the world observes.
- `is_on` — the current state, so a script can act on a fire it did not light.
