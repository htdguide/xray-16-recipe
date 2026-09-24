# src/xrNetServer/stdafx.h

> The module's shared prelude — and, incidentally, the only place the session's application
> identifier is written down.

**Needs** — [`Common/Common.hpp`](../Common/Common.hpp.md) · [`xrCore/xrCore.h`](../xrCore/xrCore.h.md) · [`NET_Shared.h`](NET_Shared.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`stdafx.cpp`](stdafx.cpp.md)
**Tier floor** — T4: it names what every file in the module needs. A rebuild in a language
with real modules deletes it.

## Purpose

A build-time convenience: one header every file in the module includes, so the common
declarations are parsed once. As a prelude it carries no decision and a rebuild discards it.

One thing inside it *is* load-bearing, and it is here only because this is where the transport
header is included.

## The application identifier

```text
CONSTANT application_id   # a 128-bit identifier, fixed for the life of the protocol
```

The server advertises it with its session and the client sends it in its discovery probe; a
discovery query carrying any other identifier is not answered. It is the coarsest possible
version gate — it separates this game's servers from every other application using the same
transport, and from a future protocol revision if anyone remembers to change it.

A rebuild needs *something* in this role and is free to choose its own value. It should
change it whenever the wire format changes, which the original never did.

A second such identifier selects a network-simulator variant of the transport's provider,
used when the engine is started with a switch that asks for artificial latency and loss. That
is a vendor-specific testing affordance; a rebuild wanting it implements impairment in its own
transport adapter.

## Notes

The prelude also suppresses one compiler diagnostic around the transport header — the vendor
header uses functions the compiler considers unsafe. Incidental in every direction.
