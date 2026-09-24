# src/xrGame/vision_client_inline.h

> The one accessor of [`vision_client`](vision_client.h.md) that is worth not going through a call for.

**Needs** — [`vision_client.h`](vision_client.h.md)
**Used by** — [`vision_client.h`](vision_client.h.md)
**Tier floor** — T3: one accessor

## Purpose

Exists only because C++ wants inline definitions outside the class body. It contributes one
accessor and no decisions; a rebuild folds it into the declaration and deletes the file.

## `visual`

**Contract** — returns the visual memory manager this sensor owns. It is never absent: the
constructor creates it and the destructor deletes it, so any caller reaching this accessor
between those two points gets a live manager. The assertion guarding that in the original
is documenting the invariant, not handling a case.
