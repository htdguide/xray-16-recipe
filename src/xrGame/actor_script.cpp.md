# src/xrGame/actor_script.cpp

> Exports the player character and the level-transition trigger to the script virtual machine.

**Needs** — [`Actor.h`](Actor.h.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`level_changer.h`](level_changer.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Registers two engine classes with the script layer. The surface is strikingly small for
the player character — one accessor — because almost everything scripts do to the player
they do through the *game object facade* the player shares with every other entity. What
the player alone has, and what this exports, is its condition: health, bleeding, radiation,
satiety, stamina, psy health.

## State

`Stateless.`

## `CActor::script_register`

**Contract** — registers two types into the script virtual machine, each as a subclass of
the game object facade and each default-constructible from script:

- the player character, exporting one accessor for its condition object;
- the level-transition trigger, exporting nothing beyond the facade — scripts need it only
  to recognize the class and to read its facade properties.

Runs once at script-engine bring-up. Names and signatures are frozen by conformance
criterion 10.

**Notes** — grouping the level-transition trigger with the player is an accident of file
placement; it has no relationship to the player beyond being small. A rebuild should
register each class where it is defined.
