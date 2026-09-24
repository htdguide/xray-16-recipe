# src/xrGame/smart_cover_object_script.cpp

> Exports the smart cover object to the script layer as a game object subtype with no members of its own.

**Needs** — [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_object.h`](smart_cover_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding declaration

## Purpose

Registers the smart cover object with the script virtual machine under the name
`smart_cover_object`, deriving from the general
[game object](../../GLOSSARY.md) facade and exposing a default constructor.

## Script surface

**Contract** — the class exposes **no methods or fields of its own**. Everything a script
can do with a placed smart cover it does through the inherited game-object surface, or
through the [smart terrain](../../GLOSSARY.md) and cover-manager interfaces elsewhere.

**Notes** — the registration exists so that a script holding a game object can identify a
smart cover by type and so that the class name is available where the spawn machinery
needs it. The exported name is part of the frozen script surface (conformance criterion
10) and cannot be renamed.
