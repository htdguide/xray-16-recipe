# src/xrGame/rat_state_base_inline.h

> Construction and the checked access to the bound rat.

**Needs** — [`rat_state_base.h`](rat_state_base.h.md)
**Used by** — [`rat_state_base.h`](rat_state_base.h.md)
**Tier floor** — T3: an accessor

## Purpose

Supplies two bodies declared in [`rat_state_base.h`](rat_state_base.h.md): a state starts
with no rat bound, and every access to the rat asserts one is bound. The accessor is
inlined because every line of every rat state goes through it.
