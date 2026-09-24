# src/xrNetServer/NET_Client.h

> Declares the client endpoint — the state queries a game layer polls, and the hooks it
> overrides to learn why a connection attempt failed.

**Needs** — [`NET_Shared.h`](NET_Shared.h.md) · [`NET_Common.h`](NET_Common.h.md) · [`NET_Server.h`](NET_Server.h.md) · [`xrCore/net_utils.h`](../xrCore/net_utils.h.md)
**Used by** — [`Level.h`](../xrGame/Level.h.md) · [`NET_Client.cpp`](NET_Client.cpp.md)
**Tier floor** — T2: it is an interface and a set of state queries. The implementation in
[`NET_Client.cpp`](NET_Client.cpp.md) is what pins the module to T1.

## Purpose

Declares the surface implemented in [`NET_Client.cpp`](NET_Client.cpp.md). The part that is
*not* repeated there, and so belongs here, is the extension contract: what a game layer must
or may supply when it builds its own client on top of this one.

## Exported units

- **`INetQueue`** — the hand-off queue of received messages. Contract in
  [`NET_Client.cpp`](NET_Client.cpp.md).
- **`IPureClient`** — the client endpoint. It is both halves of the envelope
  ([`NET_Common.h`](NET_Common.h.md)) plus the session state.

## What a game layer must supply

Nothing. Every hook has a working default, which is why a bare client compiles and connects.
What a game layer *wants* to override falls into three groups:

```text
# Refusal hooks - called from inside the connect attempt, on the connecting thread.
# Each corresponds to one way the attempt can fail, and exists so the user interface
# can say something specific instead of "could not connect".
  on_invalid_host        # the address did not resolve, or answered with no session
  on_invalid_password    # the session required a password and this one was wrong
  on_session_full        # the server is at its player limit
  on_connect_rejected    # the server refused the address outright (banned, or off-subnet)

# Session hooks
  on_session_terminate(reason : text)   # the far end ended the session; reason may be empty
  on_message(data, size)                # override to intercept before queueing

# Identity hooks
  message_name(type_id) -> text         # for tracing; default returns nothing
```

## State queries

The game layer's own state machine is driven by polling these; none of them blocks.

```text
  connect_completed()   # the server has signed on; engine messages are flowing
  connect_failed()      # the attempt is over and did not succeed
  sync_completed()      # the server-clock estimate has converged
  disconnected()        # the far end terminated the session
  client_id()           # this client's identifier, as the server knows it
  session_description() # map name, map version and download URL, read before connecting
  statistics()          # the link report
```

## Notes

The queue is drained under an explicit bracket — lock, then repeatedly peek and release, then
unlock — rather than by an iterator or a callback. The reason is that the transport delivers on
its own thread and the simulation must see a whole batch consistently. A rebuild with a
channel or an actor mailbox expresses this directly; what has to survive is that the drain is
atomic with respect to arrivals.

One build-time hook asks whether an anti-cheat client can be loaded and always answers no. It
is a vestige of an integration that was removed.
