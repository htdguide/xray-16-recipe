# src/xrGame/script_binder.h

> Declares the engine-side half of the script binding: the mixin every game object carries so a script can attach behaviour to it.

**Needs** — [`script_binder_object.h`](script_binder_object.h.md) · [`script_binder_inline.h`](script_binder_inline.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`GameObject.cpp`](GameObject.cpp.md) · [`GameObject.h`](GameObject.h.md) · [`script_binder.cpp`](script_binder.cpp.md) · [`script_binder_inline.h`](script_binder_inline.h.md)
**Tier floor** — T2: an optional reference and eleven forwarded lifecycle hooks

## Purpose

Declares the surface implemented in [`script_binder.cpp`](script_binder.cpp.md). Every game
object mixes this in, and it is what makes the engine's entity lifecycle reachable from Lua:
if a script has attached a binder, each lifecycle event is forwarded to it; if not, nothing
happens and the object behaves as pure engine.

The pairing is deliberate and worth stating once: **this is the engine side, the script side
is [`script_binder_object.h`](script_binder_object.h.md), and the attachment between them is
one optional reference set at reload time.**

## State

```text
RECORD ScriptBinder
  attached : optional<ScriptBinderObject>   # owned; none until a script attaches one
  owner    : GameObject                     # borrowed; the object this mixin belongs to
```

**Invariants** — the attached binder is owned and must be released before the owner is
destroyed; destruction asserts it is gone. Nothing may attach twice.

## Exported units

- `init` / `clear` — reset to unattached; release the attached binder if any.
- `set_object` — attach. See the implementation twin for the single-player restriction.
- `object` — read the attached binder. See [`script_binder_inline.h`](script_binder_inline.h.md).
- `reload(section)` — the attachment point: look up this object's configured binding function
  and call it. The only place a binder is ever created.
- `reinit`, `net_Spawn`, `net_Destroy`, `shedule_Update`, `save`, `load`,
  `net_SaveRelevant`, `net_Relcase` — the forwarded lifecycle hooks. Their order is the
  engine's entity lifecycle and is load-bearing; see
  [`script_binder.cpp`](script_binder.cpp.md).
