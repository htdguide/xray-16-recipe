# src/xrGame/rat_state_base.cpp

> Binds a rat state to its rat.

**Needs** — [`rat_state_base.h`](rat_state_base.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one assignment

## Purpose

The only part of the state interface that is not pure: recording which rat this state
instance belongs to. It is a source file rather than an inline because it is the one place
the binding may be checked and because the rat type need not be complete at the declaration.

## `construct`

**Contract** — records the rat. The rat must exist; a state bound to nothing would fail
later, at its first world access, with no way back to the cause.

Called once per state instance, by the manager, at the moment the state is added. The
contract and the reasoning are in [`rat_state_base.h`](rat_state_base.h.md).
