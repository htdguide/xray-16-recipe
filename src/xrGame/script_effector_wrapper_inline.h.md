# src/xrGame/script_effector_wrapper_inline.h

> Forwards the effector wrapper's construction to the effector it wraps.

**Needs** — [`script_effector_wrapper.h`](script_effector_wrapper.h.md)
**Used by** — [`script_effector_wrapper.h`](script_effector_wrapper.h.md)
**Tier floor** — T2: a constructor

## Purpose

One forwarding constructor `(type, duration)`, adding nothing. It exists only because the
wrapper is a distinct type from the effector it adapts; a rebuild in which script
subclassing needs no adapter type deletes this file outright.
