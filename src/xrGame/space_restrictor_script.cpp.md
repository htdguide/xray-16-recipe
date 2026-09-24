# src/xrGame/space_restrictor_script.cpp

> Exports the restrictor to the script virtual machine, so that scripts can recognize a restrictor game object by type.

**Needs** — [`space_restrictor.h`](space_restrictor.h.md) · [`GameObject.h`](GameObject.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one registration record

## Purpose

The registration that puts the restrictor into the script type hierarchy.

## State

`Stateless.`

## `CSpaceRestrictor::script_register`

**Contract** — registers the restrictor under the name `CSpaceRestrictor`, deriving the
game object, default-constructible from script. Runs once at script-engine bring-up. The
name is frozen by conformance criterion 10.

**Notes** — no method is exported. The registration exists purely so that the binding layer
knows the inheritance relation, which is what lets a script narrow a game object it received
to a restrictor and, more importantly, lets the many derived zone and smart-terrain exports
declare their own base. A type with no members is still a load-bearing part of the script
surface when the surface is a hierarchy.

Everything a script actually does with a restrictor — restrict an entity, add or remove
restrictions, ask whether a position is accessible — is exported on the game object facade
and on the restriction manager, not here.
