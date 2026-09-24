# src/xrGame/file_transfer.h

> Declares the two file-transfer sites — server and client — implemented in [`file_transfer.cpp`](file_transfer.cpp.md).

**Needs** — [`filetransfer_node.h`](filetransfer_node.h.md) · [`filereceiver_node.h`](filereceiver_node.h.md) · [`xrCore/Containers/AssociativeVector.hpp`](../xrCore/Containers/AssociativeVector.hpp.md) · [`xrEngine/StatGraph.h`](../xrEngine/StatGraph.h.md)
**Used by** — [`Level_network.cpp`](Level_network.cpp.md) · [`Level_network_messages.cpp`](Level_network_messages.cpp.md) · [`Level_network_start_client.cpp`](Level_network_start_client.cpp.md) · [`file_transfer.cpp`](file_transfer.cpp.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`screenshot_server.cpp`](screenshot_server.cpp.md) · [`screenshot_server.h`](screenshot_server.h.md) · [`xrServer.cpp`](xrServer.cpp.md) · [`xrServer_Connect.cpp`](xrServer_Connect.cpp.md) · [`xrServer_Disconnect.cpp`](xrServer_Disconnect.cpp.md) · [`xrServer_info.cpp`](xrServer_info.cpp.md) · [`xrServer_info.h`](xrServer_info.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the session bookkeeping for in-band file transfer. Substance is in
[`file_transfer.cpp`](file_transfer.cpp.md).

The declaration's own load-bearing content is the **keying**, which is the difference
between the two sites:

- the server keys a send by the *pair* (destination peer, source peer), because it relays —
  a file travelling from one client to another is still named by its original sender;
- the server keys a receive by the source peer alone;
- the client has one send slot and no key at all, because it only ever sends to the server;
- the client keys receives by source peer, because the server tells it whose file is
  arriving.

Exported units:

- `server_site` — the many-transfer side.
- `client_site` — the one-transfer side.
- `update_transfer` — send one chunk per active transfer, adjusting the chunk size from the
  peer's throughput.
- `on_message` — dispatch the three protocol commands. The two sites read different frames:
  the server's inbound messages carry no source field, the client's do.
- `stop_obsolete_receivers` — reap quiet receives, with a long timeout before the first
  chunk and a short one after.
- `start_transfer_file` — four source kinds on the server, two on the client. Refuses a
  duplicate key.
- `stop_transfer_file` / `stop_receive_file` — the single teardown path, which tells the peer
  whenever the session was incomplete.
- `start_receive_file` — open a receive into a file or into a caller's memory buffer.
- `is_transfer_active` / `is_receiving_active` — presence tests.
- `dbg_init_statgraph` / `dbg_update_statgraph` / `dbg_deinit_statgraph` — debug-only chunk
  size plot on the client.
