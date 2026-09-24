# src/xrGame/ai/monsters/poltergeist/poltergeist_script.cpp

> Registers the poltergeist's type name with the script layer, and nothing else.

**Needs** — [`poltergeist.h`](poltergeist.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one registration

## Purpose

Makes the poltergeist's type visible to the script layer as a subtype of the script-facing game
object, with a default constructor and no methods of its own.

Stateless.

## `script_register`

**Contract** — declares the type under the name `CPoltergeist`, derived from the script-facing
game object, constructible with no arguments. Exports no fields and no methods.

**Notes** — the registration exists so shipped scripts can *test* whether a game object is a
poltergeist and can receive one as a typed value. Everything a script actually does to a
poltergeist goes through the base game object's surface.

The exported name is one of the frozen script identifiers: shipped scripts use it and it cannot
be renamed. The separate file, and its separate precompiled header, are an artefact of the
script layer's compilation cost in the original and carry no decision.
