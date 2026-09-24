# src/xrGame/server_entity_wrapper_inline.h

> Construction and access for the server-object envelope.

**Needs** — [`server_entity_wrapper.h`](server_entity_wrapper.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3

## Purpose

Two bodies belonging to [`server_entity_wrapper.h`](server_entity_wrapper.h.md).

## Exported units

**Construction** — takes the server object to wrap, or none. The "none" case is what the
load path needs: an empty wrapper is constructed, then told to read itself from a stream,
and the read is what supplies the object. Every other construction hands ownership over
immediately.

**`object()`** — the wrapped object, with a precondition that one is present. Calling it on
an empty wrapper that has not yet been loaded is a programming error.
