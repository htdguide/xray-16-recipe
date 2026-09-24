# src/xrGame/cover_point_script.cpp

> Exports the cover point to the script virtual machine.

**Needs** — [`cover_point.h`](cover_point.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`cover_point.h`](cover_point.h.md)
**Tier floor** — T3: registration data

## Purpose

One registration entry point makes a cover point readable from Lua, so that a script driving a
creature's behaviour can be handed the cover the engine chose and reason about it — where it
is, which navigation vertex it occupies, and whether it is an authored smart cover with
animations or a plain computed spot.

## State

`Stateless.`

## `CCoverPoint::script_register`

**Contract** — registers into the script virtual machine, once at script-engine bring-up. The
type is named `cover_point` to scripts, with three readers: the position, the navigation
vertex, and whether it is a smart cover. Read-only — a script may inspect a cover point but
not construct one, because a cover point with no corresponding navigation vertex is
meaningless to everything that consumes it.

Names and signatures are frozen by conformance criterion 10.

**Notes** — the smart-cover reader is a wrapper rather than a direct binding, because the flag
is a packed bit field and the binding layer needs a whole value. That is incidental; what
matters is that the flag is part of the script surface.
