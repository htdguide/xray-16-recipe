# src/xrGame/xrServer_info.h

> Declares the one-shot uploader that pushes a server's logo and rules text to a joining client and then reports done.

**Needs** — [`xrServer_info.cpp`](xrServer_info.cpp.md) · [`file_transfer.h`](file_transfer.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`xrServer.cpp`](xrServer.cpp.md) · [`xrServer_info.cpp`](xrServer_info.cpp.md)
**Tier floor** — T2: holds raw buffers borrowed from mapped files and a completion callback

## Purpose

Declares the surface implemented in [`xrServer_info.cpp`](xrServer_info.cpp.md). One instance
serves one client at a time, and the server keeps a growable pool of them — see that file.

Exported units:

- `start_upload_info(logo, rules, to client, on complete)` — begin one upload.
- `is_active` — whether this instance is busy; how the pool picks a free one.
- `upload_server_info_callback(status, sent, total)` — the transfer layer's progress and
  completion sink.
- construction, which captures the sending client's identity, and destruction, which aborts an
  upload in flight.

## State

```text
RECORD ServerInfoUploader
  state       : ENUM { idle, uploading }
  logo, rules : borrowed byte ranges        # NOT owned; the server owns the files
  to_client   : ClientID
  from_client : ClientID                    # captured at construction: the host's client
  on_complete : callback(ClientID)
  transfers   : ref transfer site           # NOT owned
```

**Invariants** — **the two payload buffers are borrowed, not copied.** They point into the
server's open logo and rules files, which outlive every uploader. That is what lets many
simultaneous uploads share one copy of the data, and it means the server must not close those
files while any upload is active.

The completion callback is **cleared after it is invoked**, so it fires exactly once per upload.
An uploader destroyed mid-transfer still fires it, on the abort path.

**Notes** — the sending client's identity is captured at *construction* rather than at upload
time, and it is always the host's own client. It is part of the transfer layer's addressing — a
transfer is keyed by the (recipient, sender) pair — so it has to be known before a transfer can
be cancelled.

The state is an enumeration with two values and could be a flag. It is an enumeration because a
third state — waiting for the client to accept — was anticipated and never needed.
