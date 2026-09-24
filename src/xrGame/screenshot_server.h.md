# src/xrGame/screenshot_server.h

> Declares the server-side anti-cheat relay: an administrator asks a suspected client for its screen or its configuration, and the server pipes it back.

**Needs** — [`file_transfer.h`](file_transfer.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`game_cl_mp.h`](game_cl_mp.h.md) · [`screenshot_server.cpp`](screenshot_server.cpp.md) · [`xrServer.cpp`](xrServer.cpp.md) · [`xrServer_Connect.cpp`](xrServer_Connect.cpp.md) · [`xrServer_Disconnect.cpp`](xrServer_Disconnect.cpp.md)
**Tier floor** — T2: a two-legged transfer with callbacks, over a streamed file

## Purpose

A multiplayer administrator suspecting a player of cheating can demand a screenshot or a dump
of that player's configuration. Neither travels directly: the client sends it to the *server*,
which forwards it to the *administrator*. This declares the relay — one instance per
in-flight demand.

The file is named for screenshots and handles configurations identically; the two differ only
in which request byte is sent and how the result is written to disk.

## State

```text
RECORD ClientDataProxy
  admin_id      : client id           # who asked
  target_id     : client id           # who is being asked
  target_name   : text                # captured at request time, for the filename
  target_digest : text                # the target's key digest; "nulldigest" when absent
  buffer        : bytes (growable)    # the whole file, held in memory
  first_receive : bool                # the forward leg has not been started yet
  receiver      : reference to an in-progress receive
  transport     : reference to the server's file-transfer service   # borrowed
```

**Invariants**

- The file is buffered **entirely in memory** and never staged to disk on the server. A
  screenshot is bounded by the client's resolution and a configuration dump by its file set,
  so the ceiling is implicit rather than enforced. A rebuild should bound it explicitly: this
  is a remote peer filling a server-side buffer on request.
- At most one demand per target may be in flight, checked against the transfer service before
  a request is sent. There is no queue.
- The transfer service is borrowed and outlives this record. Destruction stops both legs if
  either is still running, which is the only place that cleanup happens.
- The target's name and digest are captured when the request is issued, not when the file
  arrives, so a player who disconnects mid-transfer still produces a correctly named file.

## The message kinds

```text
ENUM ClientDataEvent           # one byte, in both directions
  SCREENSHOT_REQUEST           # server to suspect: take a screenshot and send it
  CONFIGS_REQUEST              # server to suspect: dump your configuration and send it
  SCREENSHOT_RESPONSE          # server to admin: a screenshot is coming, from this player
  CONFIGS_RESPONSE             # server to admin: a configuration is coming
  SCREENSHOT_ERROR             # server to admin: it failed, with a reason
  CONFIGS_ERROR
```

Exported units: construction over the transfer service; `make_screenshot` and
`make_config_dump`, the two demands; `is_active`, whether either leg is running; and the
three transfer callbacks — screenshot received, configuration received, forward leg
progressed. The implementation is in
[`screenshot_server.cpp`](screenshot_server.cpp.md).
