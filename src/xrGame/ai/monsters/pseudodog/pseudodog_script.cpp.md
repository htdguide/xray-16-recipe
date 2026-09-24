# src/xrGame/ai/monsters/pseudodog/pseudodog_script.cpp

> Registers the three dog types with the script layer.

**Needs** — [`pseudodog.h`](pseudodog.h.md) · [`psy_dog.h`](psy_dog.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: three registrations

## Purpose

Makes the pseudodog, the psi dog and the psi dog's phantom visible to the script layer. Each is
declared as a subtype of the script-facing game object with a default constructor and no
members.

Stateless.

## `script_register` — three of them

**Contract** — three separate registrations, one per type, under the names `CAI_PseudoDog`,
`CPsyDog` and `CPsyDogPhantom`. Each exports a constructor and nothing else.

**Notes** — the three are declared as siblings derived from the game object, **not** as a
hierarchy, even though the psi dog derives from the pseudodog and the phantom from the psi dog
in the engine. So a script holding a psi dog cannot use it where a pseudodog is expected. This
flattening is deliberate in the original — the script layer only needs the types for identity
tests — and a rebuild that models the real hierarchy in script changes what shipped scripts can
do with these values.

That the phantom is exported at all is the notable one: it means a script can recognise a
conjured phantom as distinct from a real dog, which is how the game's own scripts avoid
counting phantoms as kills.

The three exported names are frozen by the shipped scripts.
