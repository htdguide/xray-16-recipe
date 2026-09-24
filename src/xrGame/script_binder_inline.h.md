# src/xrGame/script_binder_inline.h

> Reads the binder attached to a game object.

**Needs** — [`script_binder.h`](script_binder.h.md) · [`script_binder_object.h`](script_binder_object.h.md)
**Used by** — [`script_binder.h`](script_binder.h.md)
**Tier floor** — T2: a field read

## Purpose

One accessor, split out because the type it returns is only forward-declared in the header
that declares the accessor. A rebuild with no such restriction merges this into
[`script_binder.h`](script_binder.h.md) and loses nothing.

## `object`

**Contract** — returns the binder currently attached to this object, or nothing if no
script ever attached one. Never allocates, never fails.
