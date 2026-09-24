# src/xrGame/particle_params_script.cpp

> Exports the particle-placement bundle to the script virtual machine, as four constructors and nothing else.

**Needs** — [`particle_params.h`](particle_params.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

One registration entry point. Scripts construct this value and hand it to whatever plays a
particle effect; they never read it back.

## State

`Stateless.`

## `CParticleParams::script_register`

**Contract** — registers the type into the script virtual machine under the name
`particle_params`, with four constructors: no arguments, one vector, two vectors, three
vectors. No fields and no methods are exposed.

**Invariants** — the four constructors are a *progressive* signature: each additional vector
fills the next field in the order position, angles, velocity. Since the fields are not
readable from script, this overload set is the type's entire surface, and the argument order
is frozen by conformance criterion 10.

**Notes** — exposing constructors without accessors makes the value opaque to script: a script
can build one and pass it along, but cannot inspect or modify one it receives. That is
deliberate and is the minimum surface that supports the optional-tail call style the shipped
scripts use.

The binding relies on the script layer resolving overloads by argument count and type at call
time, which is one of the requirements placed on the binding seam.
