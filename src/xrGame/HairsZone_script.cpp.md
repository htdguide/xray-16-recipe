# src/xrGame/HairsZone_script.cpp

> Exports three anomaly classes to the script virtual machine.

**Needs** — [`HairsZone.h`](HairsZone.h.md) · [`AmebaZone.h`](AmebaZone.h.md) · [`NoGravityZone.h`](NoGravityZone.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

One registration entry point declares three anomaly classes to Lua. Each is registered
by its engine name, as a subclass of the game object facade, with a no-argument
constructor. That is the entire exported surface: scripts get the type for
identification and casting, and no methods beyond the game object's own.

The grouping of three unrelated anomalies into one registration function is an accident
of where someone put the code; a rebuild should register each class where it is defined.

## State

`Stateless.`

## `CHairsZone::script_register`

**Contract** — registers into the script virtual machine, under the names
`CHairsZone`, `CAmebaZone` and `CNoGravityZone`, three types each deriving from the game
object facade and each default-constructible from script. Runs once at script-engine
bring-up. Names are frozen by conformance criterion 10.
