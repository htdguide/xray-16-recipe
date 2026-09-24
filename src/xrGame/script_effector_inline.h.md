# src/xrGame/script_effector_inline.h

> Constructs a script effector of a given slot type and duration.

**Needs** — [`script_effector.h`](script_effector.h.md)
**Used by** — [`script_effector.h`](script_effector.h.md)
**Tier floor** — T2: a constructor

## Purpose

One constructor, separated only so the header can declare it before the base class's own
constructor is visible. A rebuild merges it back.

## `construct(type, duration)`

**Contract** — builds a non-looping post-process effector in the slot named by `type`, with
a lifetime of `duration` seconds, and records the slot identity locally so
[`remove`](script_effector.cpp.md) can key on it later.

**Notes**

The third argument to the base — "does this effector loop" — is fixed false. A script that
wants a permanent effect asks for a long duration and retires it explicitly, rather than
creating one the camera can never reclaim.
