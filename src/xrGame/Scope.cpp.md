# src/xrGame/Scope.cpp

> Exports the three weapon attachments to the script virtual machine.

**Needs** — [`Scope.h`](Scope.h.md) · [`Silencer.h`](Silencer.h.md) · [`GrenadeLauncher.h`](GrenadeLauncher.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — reached through its declarations in [`Scope.h`](Scope.h.md); callers name that, not this file.
**Tier floor** — T3: registration data

## Purpose

A weapon attachment is an inventory item that does nothing on its own: a scope, a silencer
and an underbarrel grenade launcher are each a carried, tradeable, weight-bearing item
whose entire effect is produced by the weapon that has it fitted. The three classes
therefore hold no behaviour, and this file is the only source any of them needs — the
declaration of all three to Lua.

Registering three unrelated classes from one of them is an accident of placement; a
rebuild should register each where it is defined.

## State

`Stateless.`

## `CScope::script_register`

**Contract** — registers three types — `CScope`, `CSilencer` and `CGrenadeLauncher` — each
deriving from the game object facade, each with a no-argument constructor and no methods.
Runs once at script-engine bring-up. The names are frozen by conformance criterion 10 and
exist so that scripts can identify an attachment in an inventory and reason about whether
a weapon can accept it.
