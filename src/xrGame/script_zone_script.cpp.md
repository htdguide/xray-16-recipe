# src/xrGame/script_zone_script.cpp

> Exports the scripted trigger volume and the smart zone to the script virtual machine as spawnable entity classes.

**Needs** — [`script_zone.h`](script_zone.h.md) · [`smart_zone.h`](smart_zone.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Two registrations, both minimal, both for the same reason: an entity class must be visible
to the script layer's object factory so that a spawn record naming it can be instantiated
and so a script can name its type. Neither class exposes any method to Lua — all their
script interaction is through callbacks, not calls.

The two registrations share a file because the smart zone is a direct specialization of
the scripted zone and the original kept their bindings together; the split is arbitrary.

## State

`Stateless.`

## `CScriptZone::script_register`

**Contract** — registers the scripted trigger volume under the name `ce_script_zone`,
derived from the factory-object interface, with a default constructor and nothing else.
The `ce_` prefix marks it as a *client entity* class in the script naming convention.

## `CSmartZone::script_register`

**Contract** — the same, under the name `ce_smart_zone`. Registered here rather than in
its own file purely by convention.
