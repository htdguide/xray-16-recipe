# src/xrGame/mincer_script.cpp

> Exports two anomaly classes to the script virtual machine.

**Needs** — [`Mincer.h`](Mincer.h.md) · [`RadioactiveZone.h`](RadioactiveZone.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Makes the mincer anomaly and the radioactive zone visible to script as types. Both are
registered here rather than in their own files because the registration is two lines and
neither class adds a script-callable method of its own.

## State

`Stateless.`

## `CMincer::script_register`

**Contract** — registers two classes into the script virtual machine, both deriving from the
game object facade and both default-constructible: the mincer — the anomaly that grinds
anything caught in it — and the radioactive zone.

Neither exposes a method. The registration exists so that script can *type-test* a game
object against these classes — "is the thing I am standing in a radioactive zone" — and so
that the class-identifier factory can construct them. A rebuild whose script layer can ask
an object for its class without a registered type needs nothing here.

**Notes** — the radioactive zone is registered from the mincer's entry point, which is
arbitrary. A rebuild registers each class from wherever it is convenient; nothing depends on
the pairing.
