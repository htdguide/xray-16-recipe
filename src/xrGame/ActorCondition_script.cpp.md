# src/xrGame/ActorCondition_script.cpp

> Exports the condition model — wounds, boosts, and every health and stamina accessor — to the script virtual machine.

**Needs** — [`ActorCondition.h`](ActorCondition.h.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`Wound.h`](Wound.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

One registration entry point declares four things to Lua: the boost descriptor, the wound
record, the shared creature condition and the actor's own. This is how a script heals the
player, poisons him, grants a temporary immunity or reads whether he is limping — the
entire survival model's script surface in one file. Names and signatures are frozen by
conformance criterion 10.

## State

`Stateless.`

## `CActorCondition::script_register`

**Contract** — registers into the script virtual machine, once at script-engine bring-up:

- **`SBooster`** — default-constructible, with three read/write fields: duration, value and
  kind. Scripts construct one and hand it to the apply call, which is why it needs a
  constructor and public fields rather than an accessor pair.
- **`CWound`** — read and write access to a single wound: its severity per damage type, its
  bleeding size, the bone it sits on, the particle bone, its healing rate, and a
  destroy flag. Also the method that adds a hit into an existing wound.
- **`CEntityCondition`** — the shared creature condition: getters and *change* operations
  (relative, not absolute) for health, stamina, radiation, psychic health, satiety, morale,
  bleeding and alcohol; the maximum-stamina pair; wound add and clear; and the identifier of
  whoever last hit this creature. The change-not-set convention is deliberate: two scripts
  adjusting the same value in one frame compose instead of overwriting.
- **The boost-kind enumeration** — seventeen values, nested inside the condition class so
  scripts address them as members. Four restoration rates, one carry limit, three
  protections and nine per-damage-type immunities.
- **`CActorCondition`**, deriving from the shared one — the seventeen single-parameter boost
  entry points, the apply and clear-all calls, the four movement restrictions, and two
  iteration helpers that call a script function once per live wound or boost and stop early
  if it returns true. The walking-weight limit is exposed as a writable field.

**Notes** — the actor's class re-exports a satiety getter the base already exports. That is
harmless duplication in the binding layer, but a rebuild resolving overloads by name should
be aware the same name appears at two levels of the hierarchy.
