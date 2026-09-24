# src/xrGame/xrServer_svclient_validation.h

> Declares the one question the ownership path asks before touching an entity: is it actually alive on the client that simulates it.

**Needs** — [`xrServer_svclient_validation.cpp`](xrServer_svclient_validation.cpp.md)
**Used by** — [`xrServer_process_event_ownership.cpp`](xrServer_process_event_ownership.cpp.md) · [`xrServer_svclient_validation.cpp`](xrServer_svclient_validation.cpp.md)
**Tier floor** — T3: one predicate

## Purpose

Declares the surface implemented in
[`xrServer_svclient_validation.cpp`](xrServer_svclient_validation.cpp.md): a single free
predicate over an entity identifier.

It is a free function rather than a method because it bridges the two halves of the process — it
is asked by the *server* about the *client* object registry — and belongs to neither.

## State

`Stateless.`
