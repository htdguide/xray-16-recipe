# src/xrGame/holder_custom.cpp

> Records who is currently riding a holder.

**Needs** — [`holder_custom.h`](holder_custom.h.md) · [`Actor.h`](Actor.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: two assignments

## Purpose

The only part of the holder interface with a shared implementation: the attachment
bookkeeping. Everything else is demanded of the implementor; see
[`holder_custom.h`](holder_custom.h.md).

## State

`Stateless` — it writes the record declared in [`holder_custom.h`](holder_custom.h.md).

## `attach_Actor`

**Contract** — records the rider, both as a generic object and, when it is one, as the player.
Always succeeds; the answer exists so an implementor can override and refuse.

**Invariants** — a holder that is already occupied is silently overwritten. The refusal
belongs in an override or in the caller's `Use` test, not here.

## `detach_Actor`

**Contract** — clears both references, which is what makes the holder unoccupied.

**Invariants** — both must be cleared together. Leaving the actor reference set on an empty
holder is a dangling reference the moment the actor is destroyed, and the occupancy test only
reads the generic one.
