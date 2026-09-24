# src/xrGame/holder_custom_script.cpp

> Exports the rideable-thing interface to script under the name `holder`.

**Needs** — [`holder_custom.h`](holder_custom.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Scripts reach a vehicle or mounted weapon through this handle rather than through its
concrete class, so one mission script can drive any rideable thing.

## State

`Stateless.`

## `CHolderCustom::script_register`

**Contract** — registers the interface as `holder` with five members:

- **`engaged`** — whether anybody is riding it.
- **`Action`** — the implementor-defined command channel, by numeric identifier and flags.
- **`SetParam`** — the three-component-vector form only.
- **`SetEnterLocked`** / **`SetExitLocked`** — seal boarding or disembarking.

**Notes** — the two-component form of the parameter setter is deliberately not exported; the
registration for it is present but disabled. Two overloads distinguished only by the arity of
a vector cannot be resolved from a dynamically typed caller, so only one may be bound. A
rebuild should give the two different names rather than reproduce the ambiguity.

The exported name is lowercase `holder` while every other class in this layer is exported
under its own type name. That is frozen: shipped scripts test for it by name.
