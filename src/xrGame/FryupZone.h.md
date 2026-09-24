# src/xrGame/FryupZone.h

> Declares the empty anomaly leaf implemented in [`FryupZone.cpp`](FryupZone.cpp.md).

**Needs** — [`script_object.h`](script_object.h.md)
**Used by** — [`FryupZone.cpp`](FryupZone.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CFryupZone`, a scriptable object subclass registered under its own class
identifier. The substance — which is almost none — is in
[`FryupZone.cpp`](FryupZone.cpp.md).

Exported units:

- `CFryupZone` — a scriptable object with no engine behaviour; a debug-only render hook.
