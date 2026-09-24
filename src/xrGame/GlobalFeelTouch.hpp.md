# src/xrGame/GlobalFeelTouch.hpp

> Declares the deny-list-only touch sense implemented in [`GlobalFeelTouch.cpp`](GlobalFeelTouch.cpp.md).

**Needs** — [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md)
**Used by** — [`GlobalFeelTouch.cpp`](GlobalFeelTouch.cpp.md) · [`Level.h`](Level.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `GlobalFeelTouch`, a participant in the touch sense that implements only the
deny half of the interface. Substance is in
[`GlobalFeelTouch.cpp`](GlobalFeelTouch.cpp.md).

Exported units:

- `GlobalFeelTouch` — a touch sense whose nearby-object set is never computed.
- `feel_touch_update` — prunes expired deny entries and does nothing else.
- `is_object_denied` — membership test on the deny list.
- the deny call itself is inherited unchanged from the base sense.
